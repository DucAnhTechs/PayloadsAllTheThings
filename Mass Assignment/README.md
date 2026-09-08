# Mass Assignment

> Tấn công Mass Assignment là một lỗ hổng bảo mật xảy ra khi một ứng dụng web tự động gán các giá trị đầu vào do người dùng cung cấp vào các thuộc tính hoặc biến của một đối tượng trong chương trình. Điều này có thể trở thành vấn đề nếu người dùng có khả năng thay đổi các thuộc tính mà họ không được phép truy cập, chẳng hạn như quyền của người dùng hoặc cờ quản trị viên (`admin flag`).

## Tóm tắt

* [Phương pháp](#methodology)
* [Các bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Phương pháp

Các lỗ hổng Mass Assignment thường xuất hiện trong các ứng dụng web sử dụng kỹ thuật hoặc hàm **Object-Relational Mapping (ORM)** để ánh xạ dữ liệu đầu vào của người dùng vào các thuộc tính của đối tượng. Trong trường hợp này, nhiều thuộc tính có thể được cập nhật cùng lúc thay vì cập nhật từng thuộc tính riêng lẻ. Nhiều framework phát triển web phổ biến như Ruby on Rails, Django và Laravel (PHP) cung cấp chức năng này.

Ví dụ, hãy xem xét một ứng dụng web sử dụng ORM và có một đối tượng người dùng với các thuộc tính `username`, `email`, `password` và `isAdmin`. Trong trường hợp thông thường, người dùng có thể cập nhật username, email và password của chính họ thông qua một form, sau đó máy chủ sẽ gán các giá trị này vào đối tượng người dùng.

Tuy nhiên, kẻ tấn công có thể cố gắng thêm tham số `isAdmin` vào dữ liệu gửi đến như sau:

```json
{
    "username": "attacker",
    "email": "attacker@email.com",
    "password": "unsafe_password",
    "isAdmin": true
}
```

Nếu ứng dụng web không kiểm tra những tham số nào được phép cập nhật theo cách này, nó có thể thiết lập thuộc tính `isAdmin` dựa trên dữ liệu do người dùng cung cấp, từ đó cấp cho kẻ tấn công quyền quản trị viên.

## Các bài lab

* [PentesterAcademy - Mass Assignment I](https://attackdefense.pentesteracademy.com/challengedetailsnoauth?cid=1964)
* [PentesterAcademy - Mass Assignment II](https://attackdefense.pentesteracademy.com/challengedetailsnoauth?cid=1922)
* [Root Me - API - Mass Assignment](https://www.root-me.org/en/Challenges/Web-Server/API-Mass-Assignment)

## Tài liệu tham khảo

* [Mass Assignment Cheat Sheet - OWASP - March 15, 2021](https://web.archive.org/web/20260216020815/https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)
* [What is Mass Assignment? Attacks and Security Tips - Yoan MONTOYA - June 15, 2023](https://www.vaadata.com/blog/what-is-mass-assignment-attacks-and-security-tips/)
