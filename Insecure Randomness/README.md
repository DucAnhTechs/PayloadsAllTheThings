# Insecure Randomness

> Insecure randomness đề cập đến các điểm yếu liên quan đến việc sinh số ngẫu nhiên trong máy tính, đặc biệt khi tính ngẫu nhiên được sử dụng cho các mục đích quan trọng về bảo mật. Các lỗ hổng trong bộ sinh số ngẫu nhiên (RNG) có thể dẫn đến output có thể dự đoán được và bị attacker khai thác, từ đó có khả năng dẫn đến data breach hoặc truy cập trái phép.

## Tóm tắt

* [Phương pháp](#methodology)
* [Seed dựa trên thời gian](#time-based-seeds)
* [GUID / UUID](#guid--uuid)

  * [Các phiên bản GUID](#guid-versions)
* [Mongo ObjectId](#mongo-objectid)
* [Uniqid](#uniqid)
* [mt_rand](#mt_rand)
* [Thuật toán tùy chỉnh](#custom-algorithms)
* [Tài liệu tham khảo](#references)

## Phương pháp

Insecure randomness xảy ra khi nguồn randomness hoặc phương pháp tạo ra các giá trị ngẫu nhiên không đủ khó dự đoán. Điều này có thể dẫn đến các output có thể dự đoán được và bị attacker khai thác. Dưới đây là các phương pháp phổ biến dễ dẫn đến insecure randomness, bao gồm seed dựa trên thời gian, GUID, UUID, MongoDB ObjectId và function `uniqid()`.

## Seed dựa trên thời gian

Nhiều bộ sinh số ngẫu nhiên (RNG) sử dụng thời gian hệ thống hiện tại (ví dụ: số milliseconds kể từ epoch) làm seed. Cách tiếp cận này có thể không an toàn vì giá trị seed có thể dễ dàng bị dự đoán, đặc biệt trong các môi trường tự động hoặc được thực thi bằng script.

```py
import random
import time

seed = int(time.time())
random.seed(seed)
print(random.randint(1, 100))
```

RNG được seed bằng thời gian hiện tại, khiến giá trị có thể dự đoán được đối với bất kỳ ai biết hoặc có thể ước tính giá trị seed.

Bằng cách biết chính xác thời gian, attacker có thể tái tạo lại giá trị random chính xác. Dưới đây là một ví dụ với thời điểm `2024-11-10 13:37`.

```python
import random
import time

# Seed dựa trên timestamp được cung cấp
seed = int(time.mktime(time.strptime('2024-11-10 13:37', '%Y-%m-%d %H:%M')))
random.seed(seed)

# Tạo số ngẫu nhiên
print(random.randint(1, 100))
```

## GUID / UUID

GUID (Globally Unique Identifier) hoặc UUID (Universally Unique Identifier) là một số 128-bit được sử dụng để xác định duy nhất thông tin trong các hệ thống máy tính. Chúng thường được biểu diễn dưới dạng chuỗi các chữ số hexadecimal, được chia thành năm nhóm và phân tách bằng dấu gạch ngang, chẳng hạn như `550e8400-e29b-41d4-a716-446655440000`.

GUID/UUID được thiết kế để có tính duy nhất trên cả không gian và thời gian, làm giảm khả năng trùng lặp ngay cả khi được tạo bởi các hệ thống khác nhau hoặc tại các thời điểm khác nhau.

### Các phiên bản GUID

Nhận diện phiên bản: `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx`

Các trường `M` 4-bit và `N` từ 1 đến 3-bit mã hóa format của chính UUID.

| Phiên bản | Ghi chú                                                                   |
| --------- | ------------------------------------------------------------------------- |
| 0         | Chỉ có `00000000-0000-0000-0000-000000000000`                             |
| 1         | Dựa trên thời gian hoặc clock sequence                                    |
| 2         | Được dành riêng trong RFC 4122 nhưng bị bỏ qua trong nhiều implementation |
| 3         | Dựa trên hash MD5                                                         |
| 4         | Được tạo ngẫu nhiên                                                       |
| 5         | Dựa trên hash SHA1                                                        |

### Công cụ

* https://github.com/intruder-io/guidtool - Công cụ để kiểm tra và tấn công GUID version 1

  ```ps1
  $ guidtool -i 95f6e264-bb00-11ec-8833-00155d01ef00
  UUID version: 1
  UUID time: 2022-04-13 08:06:13.202186
  UUID timestamp: 138691299732021860
  UUID node: 91754721024
  UUID MAC address: 00:15:5d:01:ef:00
  UUID clock sequence: 2099

  $ guidtool 1b2d78d0-47cf-11ec-8d62-0ff591f2a37c -t '2021-11-17 18:03:17' -p 10000
  ```

## Mongo ObjectId

Mongo ObjectId được tạo theo cách có thể dự đoán được. Giá trị ObjectId 12-byte bao gồm:

* **Timestamp** (4 bytes): Biểu diễn thời điểm ObjectId được tạo, được đo bằng số giây kể từ Unix epoch (January 1, 1970).
* **Machine Identifier** (3 bytes): Xác định machine nơi ObjectId được tạo. Thông thường được suy ra từ hostname hoặc IP address của machine, khiến nó có thể dự đoán được đối với các document được tạo trên cùng machine.
* **Process ID** (2 bytes): Xác định process đã tạo ObjectId. Thông thường là process ID của MongoDB server process, khiến nó có thể dự đoán được đối với các document được tạo bởi cùng process.
* **Counter** (3 bytes): Giá trị counter duy nhất được tăng lên đối với mỗi ObjectId mới được tạo. Counter được khởi tạo bằng một giá trị ngẫu nhiên khi process bắt đầu, nhưng các giá trị tiếp theo có thể dự đoán được vì chúng được tạo tuần tự.

Ví dụ về token

* `5ae9b90a2c144b9def01ec37`, `5ae9bac82c144b9def01ec39`

### Công cụ

* https://github.com/andresriancho/mongo-objectid-predict - Dự đoán Mongo ObjectId

  ```ps1
  ./mongo-objectid-predict 5ae9b90a2c144b9def01ec37
  5ae9bac82c144b9def01ec39
  5ae9bacf2c144b9def01ec3a
  5ae9bada2c144b9def01ec3b
  ```

* Python script để khôi phục `timestamp`, `process` và `counter`

  ```py
  def MongoDB_ObjectID(timestamp, process, counter):
      return "%08x%10x%06x" % (
          timestamp,
          process,
          counter,
      )

  def reverse_MongoDB_ObjectID(token):
      timestamp = int(token[0:8], 16)
      process = int(token[8:18], 16)
      counter = int(token[18:24], 16)
      return timestamp, process, counter


  def check(token):
      (timestamp, process, counter) = reverse_MongoDB_ObjectID(token)
      return token == MongoDB_ObjectID(timestamp, process, counter)

  tokens = ["5ae9b90a2c144b9def01ec37", "5ae9bac82c144b9def01ec39"]
  for token in tokens:
      (timestamp, process, counter) = reverse_MongoDB_ObjectID(token)
      print(f"{token}: {timestamp} - {process} - {counter}")
  ```

## Uniqid

Các token được tạo bằng `uniqid` dựa trên timestamp và có thể được reverse.

* [Riamse/python-uniqid](https://github.com/Riamse/python-uniqid/blob/master/uniqid.py) dựa trên timestamp
* [php/uniqid](https://github.com/php/php-src/blob/master/ext/standard/uniqid.c)

Ví dụ về token

* uniqid: `6659cea087cd6`, `6659cea087cea`
* sha256(uniqid): `4b26d474c77daf9a94d82039f4c9b8e555ad505249437c0987f12c1b80de0bf4`, `ae72a4c4cdf77f39d1b0133394c0cb24c33c61c4505a9fe33ab89315d3f5a1e4`

### Công cụ

```py
import math
import datetime

def uniqid(timestamp: float) -> str:
    sec = math.floor(timestamp)
    usec = round(1000000 * (timestamp - sec))
    return "%8x%05x" % (sec, usec)

def reverse_uniqid(value: str) -> float:
    sec = int(value[:8], 16)
    usec = int(value[8:], 16)
    return float(f"{sec}.{usec}")

tokens = ["6659cea087cd6" , "6659cea087cea"]
for token in tokens:
    t = float(reverse_uniqid(token))
    d = datetime.datetime.fromtimestamp(t)
    print(f"{token} - {t} => {d}")
```

## mt_rand

Phá `mt_rand()` bằng hai output value mà không cần bruteforce.

* https://github.com/ambionics/mt_rand-reverse - Script để khôi phục seed của `mt_rand()` chỉ với hai output và không cần bruteforce.

```ps1
./display_mt_rand.php 12345678 123
712530069 674417379

./reverse_mt_rand.py 712530069 674417379 123 1
```

## Thuật toán tùy chỉnh

Nhìn chung, không nên tự tạo thuật toán randomness của riêng mình. Dưới đây là một số ví dụ được tìm thấy trên GitHub hoặc StackOverflow đôi khi được sử dụng trong production, nhưng có thể không đáng tin cậy hoặc không an toàn.

* `$token = md5($emailId).rand(10,9999);`
* `$token = md5(time()+123456789 % rand(4000, 55000000));`

### Công cụ

Nhận diện tổng quát và sandwich attack:

* https://github.com/AethliosIK/reset-tolkien - Khai thác secret dựa trên thời gian không an toàn và triển khai Sandwich attack Resources

  ```ps1
  reset-tolkien detect 660430516ffcf -d "Wed, 27 Mar 2024 14:42:25 GMT" --prefixes "attacker@example.com" --suffixes "attacker@example.com" --timezone "-7"
  reset-tolkien sandwich 660430516ffcf -bt 1711550546.485597 -et 1711550546.505134 -o output.txt --token-format="uniqid"
  ```

## Tài liệu tham khảo

* [Breaking PHP's mt_rand() with 2 values and no bruteforce - Charles Fol - January 6, 2020](https://web.archive.org/web/20200106202157/https://www.ambionics.io/blog/php-mt-rand-prediction)
* [Cracking Time-Based Tokens: A Glimpse from a Workshop During leHACK 2025-Singularity - 4m1d0n - June 30, 2025](https://4m1d0n.github.io/retex-insecure-time-token-sandwich-attack/)
* [Exploiting Weak Pseudo-Random Number Generation in PHP’s rand and srand Functions - Jacob Moore - October 18, 2023](https://web.archive.org/web/20250919151004/https://medium.com/@moorejacob2017/exploiting-weak-pseudo-random-number-generation-in-phps-rand-and-srand-functions-445229b83e01)
* [IDOR through MongoDB Object IDs Prediction - Amey Anekar - August 25, 2020](https://web.archive.org/web/20200826103440/https://techkranti.com/idor-through-mongodb-object-ids-prediction)
* [In GUID We Trust - Daniel Thatcher - October 11, 2022](https://web.archive.org/web/20221013100900/https://www.intruder.io/research/in-guid-we-trust)
* [Multi-sandwich attack with MongoDB Object ID or the scenario for real-time monitoring of web application invitations: a new use case for the sandwich attack - Tom CHAMBARETAUD (@AethliosIK) - July 18, 2024](https://web.archive.org/web/20260201082729/https://www.aeth.cc/public/Article-Reset-Tolkien/multi-sandwich-article-en.html)
* [Secret basé sur le temps non sécurisé et attaque par sandwich - Analyse de mes recherches et publication de l’outil “Reset Tolkien” - Tom CHAMBARETAUD (@AethliosIK) - April 2, 2024](https://web.archive.org/web/20240408172738/https://www.aeth.cc/public/Article-Reset-Tolkien/secret-time-based-article-fr.html) *(FR)*
* [Unsecure time-based secret and Sandwich Attack - Analysis of my research and release of the “Reset Tolkien” tool - Tom CHAMBARETAUD (@AethliosIK) - April 2, 2024](https://web.archive.org/web/20250531084109/https://www.aeth.cc/public/Article-Reset-Tolkien/secret-time-based-article-en.html) *(EN)*
