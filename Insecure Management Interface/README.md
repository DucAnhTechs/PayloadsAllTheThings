# Insecure Management Interface

> Insecure Management Interface đề cập đến các lỗ hổng trong các giao diện quản trị được sử dụng để quản lý server, application, database hoặc network device. Các giao diện này thường kiểm soát những thiết lập nhạy cảm và có quyền truy cập mạnh vào cấu hình hệ thống, khiến chúng trở thành mục tiêu hấp dẫn đối với attacker.
> Insecure Management Interface có thể thiếu các biện pháp bảo mật phù hợp, chẳng hạn như authentication mạnh, encryption hoặc giới hạn IP, cho phép user trái phép có khả năng giành quyền kiểm soát các hệ thống quan trọng. Các vấn đề phổ biến bao gồm sử dụng default credentials, giao tiếp không được mã hóa hoặc expose giao diện ra public internet.

## Tóm tắt

* [Phương pháp](#methodology)
* [Tài liệu tham khảo](#references)

## Phương pháp

Các lỗ hổng Insecure Management Interface phát sinh khi các administrative interface của hệ thống hoặc application được bảo vệ không đúng cách, cho phép user trái phép hoặc malicious user truy cập, thay đổi cấu hình hoặc khai thác các hoạt động nhạy cảm. Những interface này thường đóng vai trò quan trọng trong việc duy trì, giám sát và kiểm soát hệ thống, do đó cần được bảo vệ nghiêm ngặt.

* Thiếu Authentication hoặc Authentication yếu:

  * Các interface có thể truy cập mà không yêu cầu credentials.
  * Sử dụng default credentials hoặc credentials yếu (ví dụ: admin/admin).

  ```ps1
  nuclei -t http/default-logins -u https://example.com
  ```

* Expose ra Public Internet

  ```ps1
  nuclei -t http/exposed-panels -u https://example.com
  nuclei -t http/exposures -u https://example.com
  ```

* Dữ liệu nhạy cảm được truyền qua HTTP dạng plain-text hoặc các protocol khác không được mã hóa.

**Ví dụ**:

* **Network Devices**: Router, switch hoặc firewall sử dụng default credentials hoặc có các lỗ hổng chưa được vá.
* **Web Applications**: Admin panel không yêu cầu authentication hoặc bị expose thông qua các URL có thể dự đoán (ví dụ: `/admin`).
* **Cloud Services**: API endpoint không có authentication phù hợp hoặc sử dụng các role có quyền quá mức.

## Tài liệu tham khảo

* [CAPEC-121: Exploit Non-Production Interfaces - CAPEC - July 30, 2020](https://web.archive.org/web/20260116113320/https://capec.mitre.org/data/definitions/121.html)
* [Exploiting Spring Boot Actuators - Michael Stepankin - February 25, 2019](https://web.archive.org/web/20250116045001/https://www.veracode.com/blog/research/exploiting-spring-boot-actuators)
* [Springboot - Official Documentation - May 9, 2024](https://web.archive.org/web/20140725032126/http://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)
