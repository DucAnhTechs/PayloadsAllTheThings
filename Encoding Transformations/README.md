# Encoding and Transformations (Mã Hóa và Biến Đổi)

> Encoding and Transformations là các kỹ thuật làm thay đổi cách dữ liệu được biểu diễn hoặc truyền tải mà không làm thay đổi ý nghĩa cốt lõi của nó. Các ví dụ phổ biến bao gồm URL encoding, Base64, HTML entity encoding, và Unicode transformations. Kẻ tấn công sử dụng các phương pháp này như những công cụ để vượt qua bộ lọc đầu vào, né tránh tường lửa ứng dụng web (WAF), hoặc thoát khỏi các quy trình khử trùng (sanitization).

## Tóm tắt

* [Unicode](#unicode)
    * [Unicode Normalization](#unicode-normalization)
    * [Punycode](#punycode)
* [Base64](#base64)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Unicode

Unicode là một tiêu chuẩn mã hóa ký tự phổ quát dùng để biểu diễn văn bản từ hầu hết mọi hệ thống chữ viết trên thế giới. Mỗi ký tự (chữ cái, số, ký hiệu, emoji) được gán một điểm mã (code point) duy nhất (ví dụ: U+0041 cho chữ "A"). Các định dạng mã hóa Unicode như UTF-8 và UTF-16 quy định cách các điểm mã này được lưu trữ dưới dạng byte.

### Unicode Normalization

Unicode normalization là quá trình chuyển đổi văn bản Unicode thành một dạng chuẩn hóa, nhất quán để các ký tự tương đương được biểu diễn giống nhau trong bộ nhớ.

[Bảng tham chiếu Unicode Normalization](https://appcheck-ng.com/wp-content/uploads/unicode_normalization.html)

* **NFC** (Normalization Form Canonical Composition): Kết hợp các chuỗi đã phân tách thành các ký tự đã tổng hợp sẵn khi có thể.
* **NFD** (Normalization Form Canonical Decomposition): Tách các ký tự thành dạng phân tách của chúng (ký tự gốc + dấu kết hợp).
* **NFKC** (Normalization Form Compatibility Composition): Giống NFC, nhưng cũng thay thế các ký tự bằng các dạng tương đương tương thích (có thể làm thay đổi hình thức/định dạng).
* **NFKD** (Normalization Form Compatibility Decomposition): Giống NFD, nhưng cũng phân tách các ký tự tương thích.

| Ký tự     | Payload               | Sau khi chuẩn hóa   |
| ------------- | --------------------- | --------------------- |
| `‥` (U+2025)  | `‥/‥/‥/etc/passwd`    | `../../../etc/passwd` |
| `︰` (U+FE30) | `︰/︰/︰/etc/passwd` | `../../../etc/passwd` |
| `＇` (U+FF07) | `＇ or ＇1＇=＇1`     | `' or '1'='1`         |
| `＂` (U+FF02) | `＂ or ＂1＂=＂1`     | `" or "1"="1`         |
| `﹣` (U+FE63) | `admin'﹣﹣`          | `admin'--`            |
| `。` (U+3002) | `domain。com`         | `domain.com`          |
| `／` (U+FF0F) | `／／domain.com`      | `//domain.com`        |
| `＜` (U+FF1C) | `＜img src=a＞`       | `<img src=a/>`        |
| `﹛` (U+FE5B) | `﹛﹛3+3﹜﹜`         | `{{3+3}}`             |
| `［` (U+FF3B) | `［［5+5］］`         | `[[5+5]]`             |
| `＆` (U+FF06) | `＆＆whoami`          | `&&whoami`            |
| `ｐ` (U+FF50) | `shell.ｐʰｐ`         | `shell.php`           |
| `ʰ` (U+02B0)  | `shell.ｐʰｐ`         | `shell.php`           |
| `ª` (U+00AA)  | `ªdmin`               | `admin`               |

```py
import unicodedata
string = "ᴾᵃʸˡᵒᵃᵈˢ𝓐𝓵𝓵𝕋𝕙𝕖𝒯𝒽𝒾𝓃ℊ𝓈"
print ('NFC: ' + unicodedata.normalize('NFC', string))
print ('NFD: ' + unicodedata.normalize('NFD', string))
print ('NFKC: ' + unicodedata.normalize('NFKC', string))
print ('NFKD: ' + unicodedata.normalize('NFKD', string))
```

### Punycode

Punycode là một cách để biểu diễn các ký tự Unicode (bao gồm chữ cái, ký hiệu và hệ chữ viết không thuộc ASCII) chỉ bằng tập hợp giới hạn các ký tự ASCII (chữ cái, chữ số và dấu gạch nối).

Nó chủ yếu được dùng trong Hệ thống Tên Miền (DNS), vốn theo truyền thống chỉ hỗ trợ ASCII. Punycode cho phép các tên miền quốc tế hóa (IDN), nhờ đó tên miền có thể chứa các ký tự từ nhiều ngôn ngữ khác nhau bằng cách chuyển đổi chúng sang dạng ASCII an toàn.

| Hiển thị trên trình duyệt (hỗ trợ IDN) | ASCII thực tế (Punycode) |
| -------------------------------- | ----------------------- |
| раypal.com                       | xn--ypal-43d9g.com      |
| paypal.com                       | paypal.com              |

Trong MySQL, các ký tự tương tự nhau được coi là bằng nhau. Hành vi này có thể bị lợi dụng trong các phần Password Reset, Forgot Password và OAuth Provider.

```sql
SELECT 'a' = 'ᵃ';
+-------------+
| 'a' = 'ᵃ'   |
+-------------+
|           1 |
+-------------+
```

Thủ thuật này hoạt động khi câu truy vấn SQL sử dụng `COLLATE utf8mb4_0900_as_cs`.

```sql
SELECT 'a' = 'ᵃ' COLLATE utf8mb4_0900_as_cs;
+----------------------------------------+
| 'a' = 'ᵃ' COLLATE utf8mb4_0900_as_cs   |
+----------------------------------------+
|                                      0 |
+----------------------------------------+
```

## Base64

Base64 encoding là một phương pháp chuyển đổi dữ liệu nhị phân (như hình ảnh hoặc tệp) hoặc văn bản có ký tự đặc biệt thành một chuỗi có thể đọc được chỉ sử dụng các ký tự ASCII (A-Z, a-z, 0-9, +, và /). Mỗi 3 byte đầu vào được chia thành 4 nhóm 6 bit và ánh xạ thành 4 ký tự Base64. Nếu đầu vào không phải là bội số của 3 byte, đầu ra sẽ được đệm thêm bằng các ký tự `=`.

```ps1
echo -n admin | base64                            
YWRtaW4=

echo -n YWRtaW4= | base64 -d
admin
```

## Labs

* [NahamCon - Puny-Code: 0-Click Account Takeover](https://github.com/VoorivexTeam/white-box-challenges/tree/main/punycode)
* [PentesterLab - Unicode and NFKC](https://pentesterlab.com/exercises/unicode-transform)

## Tài liệu tham khảo

* [Puny-Code, 0-Click Account Takeover - Voorivex - June 1, 2025](https://web.archive.org/web/20251211233427/https://blog.voorivex.team/puny-code-0-click-account-takeover)
* [Unicode normalization vulnerabilities - Lazar - September 30, 2021](https://web.archive.org/web/20251224043224/https://lazarv.com/posts/unicode-normalization-vulnerabilities/)
* [Unicode Normalization Vulnerabilities & the Special K Polyglot - AppCheck - September 2, 2019](https://web.archive.org/web/20190916002602/https://appcheck-ng.com/unicode-normalization-vulnerabilities-the-special-k-polyglot/)
* [WAF Bypassing with Unicode Compatibility - Jorge Lajara - February 19, 2020](https://web.archive.org/web/20251230185141/https://jlajara.gitlab.io/Bypass_WAF_Unicode)
* [When "Zoë" !== "Zoë". Or why you need to normalize Unicode strings - Alessandro Segala - March 11, 2019](https://web.archive.org/web/20260128220322/https://withblue.ink/2019/03/11/why-you-need-to-normalize-unicode-strings.html)
