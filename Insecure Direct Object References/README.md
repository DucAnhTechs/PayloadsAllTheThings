# Insecure Direct Object References

> Insecure Direct Object References (IDOR) là một lỗ hổng bảo mật xảy ra khi ứng dụng cho phép người dùng truy cập hoặc sửa đổi trực tiếp các object (chẳng hạn như file, bản ghi cơ sở dữ liệu hoặc URL) dựa trên input do người dùng cung cấp mà không có cơ chế kiểm soát quyền truy cập đầy đủ. Điều này có nghĩa là nếu người dùng thay đổi giá trị của một tham số (chẳng hạn như ID) trong URL hoặc API request, họ có thể truy cập hoặc thao tác với dữ liệu mà họ không được phép xem hoặc sửa đổi.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Tham số giá trị số](#numeric-value-parameter)
  * [Tham số định danh phổ biến](#common-identifiers-parameter)
  * [Bộ sinh số giả ngẫu nhiên yếu](#weak-pseudo-random-number-generator)
  * [Tham số được hash](#hashed-parameter)
  * [Tham số wildcard](#wildcard-parameter)
  * [Mẹo về IDOR](#idor-tips)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [PortSwigger/BApp Store > Authz](https://portswigger.net/bappstore/4316cc18ac5f434884b2089831c7d19e)
* [PortSwigger/BApp Store > AuthMatrix](https://portswigger.net/bappstore/30d8ee9f40c041b0bfec67441aad158e)
* [PortSwigger/BApp Store > Autorize](https://portswigger.net/bappstore/f9bbac8c4acf4aefa4d7dc92a991af2f)

## Phương pháp

IDOR là viết tắt của Insecure Direct Object Reference. Đây là một loại lỗ hổng bảo mật phát sinh khi ứng dụng cung cấp quyền truy cập trực tiếp đến các object dựa trên input do người dùng cung cấp. Kết quả là attacker có thể bypass cơ chế authorization và truy cập trực tiếp vào các resource trong hệ thống, có khả năng dẫn đến việc tiết lộ, sửa đổi hoặc xóa thông tin trái phép.

**Ví dụ về IDOR**:

Hãy tưởng tượng một web application cho phép người dùng xem profile của họ bằng cách nhấp vào link `https://example.com/profile?user_id=123`:

```php
<?php
    $user_id = $_GET['user_id'];
    $user_info = get_user_info($user_id);
    ...
```

Ở đây, `user_id=123` là một direct reference đến profile của một user cụ thể. Nếu ứng dụng không kiểm tra đúng rằng user đang đăng nhập có quyền xem profile tương ứng với `user_id=123`, attacker có thể đơn giản thay đổi tham số `user_id` để xem profile của những user khác:

```ps1
https://example.com/profile?user_id=124
```

![https://lh5.googleusercontent.com/VmLyyGH7dGxUOl60h97Lr57F7dcnDD8DmUMCZTD28BKivVI51BLPIqL0RmcxMPsmgXgvAqY8WcQ-Jyv5FhRiCBueX9Wj0HSCBhE-\_SvrDdA6\_wvDmtMSizlRsHNvTJHuy36LG47lstLpTqLK](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Insecure%20Direct%20Object%20References/Images/idor.png)

### Tham số giá trị số

Tăng hoặc giảm các giá trị này để truy cập thông tin nhạy cảm.

* Giá trị thập phân: `287789`, `287790`, `287791`, ...
* Hệ thập lục phân: `0x4642d`, `0x4642e`, `0x4642f`, ...
* Unix epoch timestamp: `1695574808`, `1695575098`, ...

**Ví dụ**:

* [HackerOne - IDOR to view User Order Information - meals](https://hackerone.com/reports/287789)
* [HackerOne - Delete messages via IDOR - naaash](https://hackerone.com/reports/697412)

### Tham số định danh phổ biến

Một số định danh có thể được đoán, chẳng hạn như tên và email, và chúng có thể cấp quyền truy cập vào dữ liệu khách hàng.

* Tên: `john`, `doe`, `john.doe`, ...
* Email: `john.doe@mail.com`
* Giá trị được mã hóa Base64: `am9obi5kb2VAbWFpbC5jb20=`

**Ví dụ**:

* [HackerOne - Insecure Direct Object Reference (IDOR) - Delete Campaigns - datph4m](https://hackerone.com/reports/1969141)

### Bộ sinh số giả ngẫu nhiên yếu

* UUID/GUID v1 có thể được dự đoán nếu biết thời điểm chúng được tạo: `95f6e264-bb00-11ec-8833-00155d01ef00`
* MongoDB Object Id được tạo theo cách có thể dự đoán được: `5ae9b90a2c144b9def01ec37`

  * Một giá trị 4-byte biểu diễn số giây kể từ Unix epoch
  * Một machine identifier 3-byte
  * Một process ID 2-byte
  * Một counter 3-byte, bắt đầu bằng một giá trị ngẫu nhiên

**Ví dụ**:

* [HackerOne - IDOR allowing to read another user's token on the Social Media Ads service - a_d_a_m](https://hackerone.com/reports/1464168)
* [IDOR through MongoDB Object IDs Prediction](https://techkranti.com/idor-through-mongodb-object-ids-prediction/)

### Tham số được hash

Đôi khi chúng ta thấy các website sử dụng các giá trị đã được hash để tạo user ID hoặc token ngẫu nhiên, chẳng hạn như `sha1(username)`, `md5(email)`, ...

* MD5: `098f6bcd4621d373cade4e832627b4f6`
* SHA1: `a94a8fe5ccb19ba61c4c0873d391e987982fbbd3`
* SHA2: `9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08`

**Ví dụ**:

* [IDOR with Predictable HMAC Generation - DiceCTF 2022 - CryptoCat](https://youtu.be/Og5_5tEg6M0)

### Tham số wildcard

Gửi wildcard (`*`, `%`, `.`, `_`) thay vì ID; một số backend có thể phản hồi dữ liệu của tất cả user.

* `GET /api/users/* HTTP/1.1`
* `GET /api/users/% HTTP/1.1`
* `GET /api/users/_ HTTP/1.1`
* `GET /api/users/. HTTP/1.1`

### Mẹo về IDOR

* Thay đổi HTTP request: `POST → PUT`
* Thay đổi content type: `XML → JSON`
* Chuyển đổi các giá trị số thành array: `{"id":19} → {"id":[19]}`
* Sử dụng Parameter Pollution: `user_id=hacker_id&user_id=victim_id`

## Labs

* [PortSwigger - Insecure Direct Object References](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

## Tài liệu tham khảo

* [From Christmas present in the blockchain to massive bug bounty - Jesse Lakerveld - March 21, 2018](http://web.archive.org/web/20180401130129/https://www.vicompany.nl/magazine/from-christmas-present-in-the-blockchain-to-massive-bug-bounty)
* [How-To: Find IDOR (Insecure Direct Object Reference) Vulnerabilities for large bounty rewards - Sam Houton - November 9, 2017](https://web.archive.org/web/20260221194813/https://www.bugcrowd.com/blog/how-to-find-idor-insecure-direct-object-reference-vulnerabilities-for-large-bounty-rewards/)
* [Hunting Insecure Direct Object Reference Vulnerabilities for Fun and Profit (PART-1) - Mohammed Abdul Raheem - February 2, 2018](https://web.archive.org/web/20190509043727/https://codeburst.io/hunting-insecure-direct-object-reference-vulnerabilities-for-fun-and-profit-part-1-f338c6a52782)
* [IDOR - how to predict an identifier? Bug bounty case study - Bug Bounty Reports Explained - September 21, 2023](https://web.archive.org/web/20231027235449/https://youtu.be/wx5TwS0Dres)
* [Insecure Direct Object Reference Prevention Cheat Sheet - OWASP - July 31, 2023](https://web.archive.org/web/20140316052400/https://www.owasp.org/index.php/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet)
* [Insecure direct object references (IDOR) - PortSwigger - December 25, 2019](https://web.archive.org/web/20260301072233/https://portswigger.net/web-security/access-control/idor)
* [Testing for IDORs - PortSwigger - October 29, 2024](https://web.archive.org/web/20230604162333/https://portswigger.net/burp/documentation/desktop/testing-workflow/access-controls/testing-for-idors)
* [Testing for Insecure Direct Object References (OTG-AUTHZ-004) - OWASP - August 8, 2014](https://web.archive.org/web/20170712205114/https://www.owasp.org/index.php/Testing_for_Insecure_Direct_Object_References_%28OTG-AUTHZ-004%29)
* [The Rise of IDOR - HackerOne - April 2, 2021](https://web.archive.org/web/20211004153030/https://www.hackerone.com/company-news/rise-idor)
* [Web to App Phone Notification IDOR to view Everyone's Airbnb Messages - Brett Buerhaus - March 31, 2017](https://web.archive.org/web/20170408053950/http://buer.haus:80/2017/03/31/airbnb-web-to-app-phone-notification-idor-to-view-everyones-messages)
