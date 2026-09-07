# DNS Rebinding

> DNS rebinding thay đổi địa chỉ IP của một tên máy chủ do kẻ tấn công kiểm soát thành địa chỉ IP của ứng dụng mục tiêu, qua đó vượt qua [chính sách cùng nguồn gốc (same-origin policy)](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) và cho phép trình duyệt gửi các yêu cầu tùy ý đến ứng dụng mục tiêu cũng như đọc được phản hồi của chúng.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
* [Vượt Qua Các Cơ Chế Bảo Vệ](#protection-bypasses)
    * [0.0.0.0](#0000)
    * [CNAME](#cname)
    * [localhost](#localhost)
* [Tài Liệu Tham Khảo](#references)

## Công cụ

* [nccgroup/singularity](https://github.com/nccgroup/singularity) - Một framework tấn công DNS rebinding.
* [rebind.it](http://rebind.it/) - Singularity of Origin Web Client.
* [taviso/rbndr](https://github.com/taviso/rbndr) - Dịch vụ DNS Rebinding đơn giản
* [taviso/rebinder](https://lock.cmpxchg8b.com/rebinder.html) - Công cụ hỗ trợ rbndr

## Phương pháp

**Giai đoạn thiết lập**:

* Đăng ký một tên miền độc hại (ví dụ: `malicious.com`).
* Cấu hình một máy chủ DNS tùy chỉnh có khả năng phân giải `malicious.com` sang các địa chỉ IP khác nhau.

**Tương tác ban đầu với nạn nhân**:

* Tạo một trang web trên `malicious.com` chứa JavaScript độc hại hoặc một cơ chế khai thác khác.
* Dụ nạn nhân truy cập vào trang web độc hại đó (ví dụ: thông qua lừa đảo (phishing), kỹ thuật tấn công phi kỹ thuật (social engineering), hoặc quảng cáo).

**Phân giải DNS ban đầu**:

* Khi trình duyệt của nạn nhân truy cập `malicious.com`, nó sẽ truy vấn máy chủ DNS của kẻ tấn công để lấy địa chỉ IP.
* Máy chủ DNS phân giải `malicious.com` thành một địa chỉ IP ban đầu, trông có vẻ hợp lệ (ví dụ: 203.0.113.1).

**Rebinding sang IP nội bộ**:

* Sau yêu cầu ban đầu của trình duyệt, máy chủ DNS của kẻ tấn công sẽ cập nhật kết quả phân giải cho `malicious.com` thành một địa chỉ IP riêng tư hoặc nội bộ (ví dụ: 192.168.1.1, tương ứng với router của nạn nhân hoặc các thiết bị nội bộ khác).

Điều này thường đạt được bằng cách đặt thời gian TTL (time-to-live) rất ngắn cho phản hồi DNS ban đầu, buộc trình duyệt phải phân giải lại tên miền.

**Khai thác cùng nguồn gốc:**

Trình duyệt coi các phản hồi tiếp theo là đến từ cùng một nguồn gốc (`malicious.com`).

JavaScript độc hại đang chạy trong trình duyệt của nạn nhân giờ đây có thể gửi yêu cầu đến các địa chỉ IP nội bộ hoặc dịch vụ cục bộ (ví dụ: 192.168.1.1 hoặc 127.0.0.1), qua đó vượt qua các giới hạn của chính sách cùng nguồn gốc.

**Ví dụ:**

1. Đăng ký một tên miền.
2. [Thiết lập Singularity of Origin](https://github.com/nccgroup/singularity/wiki/Setup-and-Installation).
3. Chỉnh sửa [trang HTML autoattack](https://github.com/nccgroup/singularity/blob/master/html/autoattack.html) theo nhu cầu của bạn.
4. Truy cập `http://rebinder.your.domain:8080/autoattack.html`.
5. Chờ cuộc tấn công hoàn tất (có thể mất vài giây/phút).

## Vượt Qua Các Cơ Chế Bảo Vệ

> Hầu hết các cơ chế bảo vệ DNS được triển khai dưới dạng chặn các phản hồi DNS chứa địa chỉ IP không mong muốn tại vành đai mạng, khi phản hồi DNS đi vào mạng nội bộ. Hình thức bảo vệ phổ biến nhất là chặn các địa chỉ IP riêng tư theo định nghĩa trong RFC 1918 (tức là 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16). Một số công cụ cho phép chặn thêm localhost (127.0.0.0/8), mạng nội bộ (local), hoặc dải mạng 0.0.0.0/0.

Trong trường hợp cơ chế bảo vệ DNS được bật (thường mặc định là tắt), NCC Group đã ghi lại nhiều [cách vượt qua bảo vệ DNS](https://github.com/nccgroup/singularity/wiki/Protection-Bypasses) có thể sử dụng.

### 0.0.0.0

Ta có thể dùng địa chỉ IP 0.0.0.0 để truy cập localhost (127.0.0.1) nhằm vượt qua các bộ lọc chặn phản hồi DNS chứa 127.0.0.1 hoặc 127.0.0.0/8.

### CNAME

Ta có thể dùng bản ghi DNS CNAME để vượt qua giải pháp bảo vệ DNS chặn tất cả các địa chỉ IP nội bộ.
Vì phản hồi của chúng ta chỉ trả về một CNAME của máy chủ nội bộ,
quy tắc lọc địa chỉ IP nội bộ sẽ không được áp dụng.
Sau đó, máy chủ DNS nội bộ cục bộ sẽ phân giải CNAME đó.

```bash
$ dig cname.example.com +noall +answer
; <<>> DiG 9.11.3-1ubuntu1.15-Ubuntu <<>> example.com +noall +answer
;; global options: +cmd
cname.example.com.            381     IN      CNAME   target.local.
```

### localhost

Ta có thể dùng "localhost" làm bản ghi DNS CNAME để vượt qua các bộ lọc chặn phản hồi DNS chứa 127.0.0.1.

```bash
$ dig www.example.com +noall +answer
; <<>> DiG 9.11.3-1ubuntu1.15-Ubuntu <<>> example.com +noall +answer
;; global options: +cmd
localhost.example.com.            381     IN      CNAME   localhost.
```

## Tài Liệu Tham Khảo

* [How Do DNS Rebinding Attacks Work? - NCC Group - April 9, 2019](https://github.com/nccgroup/singularity/wiki/How-Do-DNS-Rebinding-Attacks-Work%3F)
