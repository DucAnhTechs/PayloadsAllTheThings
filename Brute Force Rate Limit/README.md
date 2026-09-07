# Brute Force & Giới hạn tần suất (Rate Limit)

## Mục lục

* [Công cụ](#tools)
* [Bruteforce](#bruteforce)
    * [Burp Suite Intruder](#burp-suite-intruder)
    * [FFUF](#ffuf)
* [Giới hạn tần suất](#rate-limit)
    * [TLS Stack - JA3](#tls-stack---ja3)
    * [Mạng IPv4](#network-ipv4)
    * [Mạng IPv6](#network-ipv6)
* [Tài liệu tham khảo](#references)

## Công cụ

* [ZephrFish/OmniProx](https://github.com/ZephrFish/OmniProx) - Xoay vòng IP từ nhiều nhà cung cấp khác nhau - Giống như FireProx nhưng cho GCP, Azure, Alibaba và CloudFlare.
* [ddd/gpb](https://github.com/ddd/gpb) - Brute-force số điện thoại của bất kỳ người dùng Google nào trong khi xoay vòng các địa chỉ IPv6.
* [ffuf/ffuf](https://github.com/ffuf/ffuf) - Công cụ fuzzing web nhanh được viết bằng Go.
* [PortSwigger/Burp Suite](https://portswigger.net/burp) - Nền tảng bảo mật ứng dụng web, kiểm thử xâm nhập, và quét lỗ hổng hàng đầu.
* [lwthiker/curl-impersonate](https://github.com/lwthiker/curl-impersonate) - Một bản build đặc biệt của curl có thể giả mạo Chrome & Firefox.

## Bruteforce

Trong ngữ cảnh web, brute-force đề cập đến phương pháp cố gắng truy cập trái phép vào các ứng dụng web, đặc biệt thông qua các form đăng nhập hoặc các trường nhập liệu khác của người dùng. Kẻ tấn công nhập một cách có hệ thống nhiều tổ hợp thông tin xác thực hoặc các giá trị khác (ví dụ: lặp qua các dải số) để khai thác mật khẩu yếu hoặc các biện pháp bảo mật không đầy đủ.

Chẳng hạn, họ có thể gửi hàng nghìn tổ hợp username và password hoặc đoán các security token bằng cách lặp qua một dải, ví dụ từ 0 đến 10.000. Phương pháp này có thể dẫn đến truy cập trái phép và rò rỉ dữ liệu nếu không được giảm thiểu hiệu quả.

Các biện pháp đối phó như giới hạn tần suất (rate limiting), chính sách khóa tài khoản, CAPTCHA, và yêu cầu mật khẩu mạnh là cần thiết để bảo vệ ứng dụng web khỏi các cuộc tấn công brute-force như vậy.

### Burp Suite Intruder

* **Sniper attack**: nhắm vào một vị trí duy nhất (một biến) trong khi lặp qua một bộ payload.

    ```ps1

    Username: password
    Username1:Password1
    Username1:Password2
    Username1:Password3
    Username1:Password4
    ```

* **Battering ram attack**: gửi cùng một payload đến tất cả các vị trí đã đánh dấu cùng một lúc bằng cách sử dụng một bộ payload duy nhất.

    ```ps1
    Username1:Username1
    Username2:Username2
    Username3:Username3
    Username4:Username4
    ```

* **Pitchfork attack**: sử dụng nhiều danh sách payload khác nhau song song, kết hợp mục thứ n từ mỗi danh sách vào một request.

    ```ps1
    Username1:Password1
    Username2:Password2
    Username3:Password3
    Username4:Password4
    ```

* **Cluster bomb attack**: lặp qua tất cả các tổ hợp của nhiều bộ payload.

    ```ps1
    Username1:Password1
    Username1:Password2
    Username1:Password3
    Username1::Password4

    Username2:Password1
    Username2:Password2
    Username2:Password3
    Username2:Password4
    ```

### FFUF

```bash
ffuf -w usernames.txt:USER -w passwords.txt:PASS \
     -u https://target.tld/login \
     -X POST -d "username=USER&password=PASS" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -H "X-Forwarded-For: FUZZ" -w ipv4-list.txt:FUZZ \
     -mc all
```

## Giới hạn tần suất

### HTTP Pipelining

HTTP pipelining là một tính năng của HTTP/1.1 cho phép client gửi nhiều HTTP request trên một kết nối TCP liên tục (persistent) duy nhất mà không cần chờ các phản hồi tương ứng trước. Client "xếp hàng" (pipes) các request lần lượt nối tiếp nhau trên cùng một kết nối.

### TLS Stack - JA3

JA3 là một phương pháp để lấy dấu vân tay (fingerprinting) các TLS client (và JA3S cho TLS server) bằng cách băm (hashing) nội dung của các thông điệp TLS "hello". Nó cung cấp một định danh gọn nhẹ mà bạn có thể sử dụng để phát hiện, phân loại, và theo dõi client trên mạng ngay cả khi các trường giao thức cấp cao hơn (như HTTP user-agent) bị ẩn hoặc giả mạo.

> JA3 thu thập các giá trị thập phân của các byte cho các trường sau trong gói tin Client Hello: SSL Version, Accepted Ciphers, List of Extensions, Elliptic Curves, và Elliptic Curve Formats. Sau đó nó nối các giá trị này lại với nhau theo thứ tự, sử dụng dấu "," để phân tách mỗi trường và dấu "-" để phân tách mỗi giá trị trong mỗi trường.

* JA3 của Burp Suite: `53d67b2a806147a7d1d5df74b54dd049`, `62f6a6727fda5a1104d5b147cd82e520`
* JA3 của Tor Client: `e7d705a3286e19ea42f587b344ee6865`

**Biện pháp đối phó:**

* Sử dụng công cụ tự động hóa điều khiển trình duyệt (Puppeteer / Playwright)
* Giả mạo bắt tay TLS (TLS handshake) bằng [lwthiker/curl-impersonate](https://github.com/lwthiker/curl-impersonate)
* Các plugin ngẫu nhiên hóa JA3 cho trình duyệt/thư viện

### Mạng IPv4

Sử dụng nhiều proxy để giả lập nhiều client.

```bash
proxychains ffuf -w wordlist.txt -u https://target.tld/FUZZ
```

* Sử dụng `random_chain` để xoay vòng mỗi request

    ```ps1
    random_chain
    ```

* Đặt số lượng proxy được xích (chain) trên mỗi kết nối là 1.

    ```ps1
    chain_len = 1
    ```

* Cuối cùng, chỉ định các proxy trong một file cấu hình:

    ```ps1
    # type  host      port
    socks5  127.0.0.1 1080
    socks5  192.168.1.50 1080
    http    proxy1.example.com 8080
    http    proxy2.example.com 8080
    ```

### Mạng IPv6

Nhiều nhà cung cấp dịch vụ đám mây, chẳng hạn như Vultr, cung cấp các dải IPv6 /64, mang lại một lượng địa chỉ khổng lồ (18.446.744.073.709.551.616). Điều này cho phép xoay vòng IP trên diện rộng trong các cuộc tấn công brute-force.

## Tài liệu tham khảo

* [Bruteforcing the phone number of any Google user - brutecat - June 9, 2025](https://web.archive.org/web/20250609141236/https://brutecat.com/articles/leaking-google-phones)
* [Burp Intruder attack types - PortSwigger - August 19, 2025](https://web.archive.org/web/20260124024947/https://portswigger.net/burp/documentation/desktop/tools/intruder/configure-attack/attack-types)
* [Detecting and annoying Burp users - Julien Voisin - May 3, 2021](https://web.archive.org/web/20260102160139/https://dustri.org/b/detecting-and-annoying-burp-users.html)
* [OmniProx: Multi-Cloud IP Rotation Made Simple - Andy Gill - September 28, 2025](https://web.archive.org/web/20260215082718/https://blog.zsec.uk/omniprox/)
