# Insecure Source Code Management

> Source Code Management (SCM) không an toàn có thể dẫn đến nhiều lỗ hổng nghiêm trọng trong các ứng dụng và dịch vụ web. Developer thường sử dụng các hệ thống SCM như Git và Subversion (SVN) để quản lý các phiên bản source code. Tuy nhiên, các thực hành bảo mật kém, chẳng hạn như để lộ các thư mục `.git` và `.svn` trong môi trường production và cho phép chúng có thể được truy cập từ Internet, có thể tạo ra những rủi ro đáng kể.

## Tóm tắt

* [Phương pháp](#methodology)

  * [Bazaar](./Bazaar.md)
  * [Git](./Git.md)
  * [Mercurial](./Mercurial.md)
  * [Subversion](./Subversion.md)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Phương pháp

Việc để lộ các thư mục của hệ thống quản lý phiên bản trên web server có thể dẫn đến những rủi ro bảo mật nghiêm trọng, bao gồm:

* **Rò rỉ Source Code** : Attacker có thể tải xuống toàn bộ source code của repository, từ đó có được quyền truy cập vào logic của ứng dụng.
* **Lộ thông tin nhạy cảm** : Secrets được nhúng trong code, các file cấu hình và credentials có thể tồn tại bên trong codebase.
* **Lộ lịch sử Commit** : Attacker có thể xem các thay đổi trong quá khứ, từ đó phát hiện những thông tin nhạy cảm đã từng bị lộ và sau đó được khắc phục.

Bước đầu tiên là thu thập thông tin về ứng dụng mục tiêu. Việc này có thể được thực hiện bằng nhiều công cụ và kỹ thuật web reconnaissance khác nhau.

* **Kiểm tra thủ công** : Kiểm tra URL thủ công bằng cách truy cập các đường dẫn SCM phổ biến.

  * Git: `http://target.com/.git/`
  * SVN: `http://target.com/.svn/`

* **Công cụ tự động** : Tham khảo trang tương ứng với công nghệ cụ thể.

Sau khi xác định được một thư mục SCM tiềm năng, hãy kiểm tra HTTP response code và nội dung trả về. Có thể cần bypass các rule của `.htaccess` hoặc Reverse Proxy.

Rule NGINX bên dưới trả về response `403 (Forbidden)` thay vì `404 (Not Found)` khi truy cập endpoint `/.git`.

```ps1
location /.git {
  deny all;
}
```

Ví dụ với Git, kỹ thuật khai thác không yêu cầu phải liệt kê được nội dung của thư mục `.git` (`http://target.com/.git/`); việc trích xuất dữ liệu vẫn có thể được thực hiện nếu các file có thể được đọc.

## Labs

* [Root Me - Quản lý Code Không An Toàn](https://www.root-me.org/fr/Challenges/Web-Serveur/Insecure-Code-Management)

## Tài liệu tham khảo

* [Thư mục và file ẩn như một nguồn thông tin nhạy cảm về ứng dụng web - bl4de - April 30, 2017](https://github.com/bl4de/research/tree/master/hidden_directories_leaks)
