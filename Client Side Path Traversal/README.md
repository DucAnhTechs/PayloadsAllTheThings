# Client Side Path Traversal

> Client-Side Path Traversal (CSPT), đôi khi còn được gọi là "On-site Request Forgery", là một lỗ hổng có thể bị khai thác như một công cụ cho các cuộc tấn công CSRF hoặc XSS.  
> Nó lợi dụng khả năng của phía client trong việc thực hiện các request bằng fetch tới một URL, trong đó nhiều ký tự "../" có thể được chèn vào. Sau khi được chuẩn hóa (normalization), các ký tự này sẽ chuyển hướng request đến một URL khác, có thể dẫn đến các lỗ hổng bảo mật.  
> Vì mọi request đều được khởi tạo từ bên trong frontend của ứng dụng, trình duyệt sẽ tự động đính kèm cookie và các cơ chế xác thực khác, khiến chúng có thể bị khai thác trong các cuộc tấn công này.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [CSPT to XSS](#cspt-to-xss)
    * [CSPT to CSRF](#cspt-to-xss)
* [Bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [doyensec/CSPTBurpExtension](https://github.com/doyensec/CSPTBurpExtension) - CSPT là một extension mã nguồn mở cho Burp Suite dùng để tìm và khai thác Client-Side Path Traversal.

## Phương pháp

### CSPT to XSS

![cspt-query-param](https://matanber.com/images/blog/cspt-query-param.png)

Một trang được phục vụ sau đó gọi hàm fetch, gửi một request đến một URL với dữ liệu đầu vào do kẻ tấn công kiểm soát mà không được mã hóa đúng cách trong đường dẫn (path), cho phép kẻ tấn công chèn các chuỗi `../` vào đường dẫn và khiến request được gửi đến một endpoint tùy ý. Hành vi này được gọi là lỗ hổng CSPT.

**Ví dụ**:

* Trang `https://example.com/static/cms/news.html` nhận tham số `newsitemid`
* Sau đó fetch nội dung của `https://example.com/newitems/<newsitemid>`
* Một lỗi chèn văn bản (text injection) cũng được phát hiện tại `https://example.com/pricing/default.js` thông qua tham số `cb`
* Payload cuối cùng là `https://example.com/static/cms/news.html?newsitemid=../pricing/default.js?cb=alert(document.domain)//`

### CSPT to CSRF

CSPT sẽ chuyển hướng các HTTP request hợp lệ, cho phép frontend thêm các token cần thiết cho các lệnh gọi API, chẳng hạn như token xác thực hoặc CSRF. Khả năng này có thể bị khai thác để vượt qua các biện pháp bảo vệ CSRF hiện có.

|                                             | CSRF               | CSPT2CSRF          |
| ------------------------------------------- | -----------------  | ------------------ |
| POST CSRF ?                                 | :white_check_mark: | :white_check_mark: |
| Có thể kiểm soát body không ?               | :white_check_mark: | :x:                |
| Có thể hoạt động với anti-CSRF token không ? | :x:                | :white_check_mark: |
| Có thể hoạt động với Samesite=Lax không ?   | :x:                | :white_check_mark: |
| GET / PATCH / PUT / DELETE CSRF ?           | :x:                | :white_check_mark: |
| CSRF chỉ với 1 cú click ?                   | :x:                | :white_check_mark: |
| Mức độ ảnh hưởng có phụ thuộc vào source và sink không ? | :x:                | :white_check_mark: |

Các kịch bản thực tế:

* CSPT2CSRF chỉ với 1 cú click trong Rocket.Chat
* CVE-2023-45316: CSPT2CSRF với sink POST trong Mattermost : `/<team>/channels/channelname?telem_action=under_control&forceRHSOpen&telem_run_id=../../../../../../api/v4/caches/invalidate`
* CVE-2023-6458: CSPT2CSRF với sink GET trong Mattermost
* [Client Side Path Manipulation - erasec.be](https://www.erasec.be/blog/client-side-path-manipulation/): CSPT2CSRF `https://example.com/signup/invite?email=foo%40bar.com&inviteCode=123456789/../../../cards/123e4567-e89b-42d3-a456-556642440000/cancel?a=`
* [CVE-2023-5123 : CSPT2CSRF trong plugin JSON API của Grafana](https://medium.com/@maxime.escourbiac/grafana-cve-2023-5123-write-up-74e1be7ef652)

## Bài lab

* [doyensec/CSPTPlayground](https://github.com/doyensec/CSPTPlayground) - CSPTPlayground là một playground mã nguồn mở để tìm và khai thác Client-Side Path Traversal (CSPT).
* [Root Me - CSPT - The Ruler](https://www.root-me.org/en/Challenges/Web-Client/CSPT-The-Ruler)

## Tài liệu tham khảo

* [Exploiting Client-Side Path Traversal to Perform Cross-Site Request Forgery - Introducing CSPT2CSRF - Maxence Schmitt - July 2, 2024](https://web.archive.org/web/20260222183040/https://blog.doyensec.com/2024/07/02/cspt2csrf.html)
* [Exploiting Client-Side Path Traversal - CSRF is dead, long live CSRF - Whitepaper - Maxence Schmitt - July 2, 2024](https://web.archive.org/web/20240702212818/https://www.doyensec.com/resources/Doyensec_CSPT2CSRF_Whitepaper.pdf)
* [Exploiting Client-Side Path Traversal - CSRF is Dead, Long Live CSRF - OWASP Global AppSec 2024 - Maxence Schmitt - June 24, 2024](https://web.archive.org/web/20250521192653/https://www.doyensec.com/resources/Doyensec_CSPT2CSRF_OWASP_Appsec_Lisbon.pdf)
* [Leaking Jupyter instance auth token chaining CVE-2023-39968, CVE-2024-22421 and a chromium bug - Davwwwx - August 30, 2023](https://web.archive.org/web/20240703155707/https://blog.xss.am/2023/08/cve-2023-39968-jupyter-token-leak/)
* [On-site request forgery - Dafydd Stuttard - May 3, 2007](https://web.archive.org/web/20260212042947/https://portswigger.net/blog/on-site-request-forgery)
* [Bypassing WAFs to Exploit CSPT Using Encoding Levels - Matan Berson - May 10, 2024](https://web.archive.org/web/20240512110749/https://matanber.com/blog/cspt-levels)
* [Automating Client-Side Path Traversals Discovery - Vitor Falcao - October 3, 2024](https://web.archive.org/web/20241004042613/https://vitorfalcao.com/posts/automating-cspt-discovery/)
* [CSPT the Eval Villain Way! - Dennis Goodlett - December 3, 2024](https://web.archive.org/web/20241203171704/https://blog.doyensec.com/2024/12/03/cspt-with-eval-villain.html)
* [Bypassing File Upload Restrictions To Exploit Client-Side Path Traversal - Maxence Schmitt - January 9, 2025](https://web.archive.org/web/20250109093347/https://blog.doyensec.com/2025/01/09/cspt-file-upload.html)
