# MFA Bypasses

> Xác thực đa yếu tố (MFA) là một biện pháp bảo mật yêu cầu người dùng cung cấp từ hai yếu tố xác minh trở lên để truy cập vào một hệ thống, ứng dụng hoặc mạng. Nó kết hợp thứ người dùng biết (chẳng hạn như mật khẩu), thứ người dùng có (chẳng hạn như điện thoại hoặc security token), và/hoặc thứ thuộc về người dùng (xác minh sinh trắc học). Phương pháp nhiều lớp này tăng cường bảo mật bằng cách khiến việc truy cập trái phép trở nên khó khăn hơn, ngay cả khi mật khẩu đã bị lộ.
> MFA Bypasses là các kỹ thuật mà attacker sử dụng để vượt qua cơ chế bảo vệ MFA. Những phương pháp này có thể bao gồm khai thác điểm yếu trong quá trình triển khai MFA, chặn các authentication token, sử dụng social engineering để thao túng người dùng hoặc nhân viên hỗ trợ, hoặc khai thác các lỗ hổng liên quan đến session.

## Tóm tắt

* [Thao túng Response](#response-manipulation)
* [Thao túng Status Code](#status-code-manipulation)
* [Rò rỉ mã 2FA trong Response](#2fa-code-leakage-in-response)
* [Phân tích file JS](#js-file-analysis)
* [Khả năng tái sử dụng mã 2FA](#2fa-code-reusability)
* [Thiếu cơ chế bảo vệ Brute-Force](#lack-of-brute-force-protection)
* [Thiếu kiểm tra tính toàn vẹn của mã 2FA](#missing-2fa-code-integrity-validation)
* [CSRF khi vô hiệu hóa 2FA](#csrf-on-2fa-disabling)
* [Password Reset vô hiệu hóa 2FA](#password-reset-disable-2fa)
* [Lạm dụng Backup Code](#backup-code-abuse)
* [Clickjacking trên trang vô hiệu hóa 2FA](#clickjacking-on-2fa-disabling-page)
* [Bật 2FA không làm hết hạn các Session đang hoạt động trước đó](#enabling-2fa-doesnt-expire-previously-active-sessions)
* [Bypass 2FA bằng Force Browsing](#bypass-2fa-by-force-browsing)
* [Bypass 2FA bằng null hoặc 000000](#bypass-2fa-with-null-or-000000)
* [Bypass 2FA bằng array](#bypass-2fa-with-array)

## Bypass 2FA

### Thao túng Response

Nếu response là `"success":false`

Thay đổi thành `"success":true`

### Thao túng Status Code

Nếu Status Code là **4xx**

Thử thay đổi thành **200 OK** và kiểm tra xem có thể vượt qua restriction hay không.

### Rò rỉ mã 2FA trong Response

Kiểm tra response của request kích hoạt mã 2FA để tìm mã bị rò rỉ.

### Phân tích file JS

Trường hợp này hiếm gặp, nhưng một số file JS có thể chứa thông tin về mã 2FA, vì vậy đáng để kiểm tra.

### Khả năng tái sử dụng mã 2FA

Cùng một mã có thể được sử dụng lại.

### Thiếu cơ chế bảo vệ Brute-Force

Có thể brute-force mã 2FA với bất kỳ độ dài nào.

### Thiếu kiểm tra tính toàn vẹn của mã 2FA

Mã của bất kỳ user account nào cũng có thể được sử dụng để bypass 2FA.

### CSRF khi vô hiệu hóa 2FA

Không có CSRF Protection khi vô hiệu hóa 2FA, đồng thời cũng không có bước xác nhận authentication.

### Password Reset vô hiệu hóa 2FA

2FA bị vô hiệu hóa khi thay đổi password/email.

### Lạm dụng Backup Code

Bypass 2FA bằng cách lạm dụng tính năng Backup Code.

Sử dụng các kỹ thuật đã đề cập ở trên để bypass Backup Code nhằm xóa/reset các restriction của 2FA.

### Clickjacking trên trang vô hiệu hóa 2FA

Nhúng trang vô hiệu hóa 2FA bằng iframe và sử dụng social engineering để khiến nạn nhân vô hiệu hóa 2FA.

### Bật 2FA không làm hết hạn các Session đang hoạt động trước đó

Nếu session đã bị hijack và tồn tại session timeout vulnerability.

### Bypass 2FA bằng Force Browsing

Nếu ứng dụng redirect đến URL `/my-account` sau khi đăng nhập khi 2FA bị vô hiệu hóa, hãy thử thay `/2fa/verify` bằng `/my-account` khi 2FA được bật để bypass bước xác minh.

### Bypass 2FA bằng null hoặc 000000

Nhập mã **000000** hoặc **null** để bypass cơ chế bảo vệ 2FA.

### Bypass 2FA bằng array

```json
{
    "otp":[
        "1234",
        "1111",
        "1337", // GOOD OTP
        "2222",
        "3333",
        "4444",
        "5555"
    ]
}
```
