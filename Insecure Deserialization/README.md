# Insecure Deserialization

> Serialization là quá trình chuyển đổi một object thành một định dạng dữ liệu có thể được khôi phục lại sau này. Người ta thường serialize object để lưu trữ hoặc gửi chúng trong quá trình giao tiếp. Deserialization là quá trình ngược lại — lấy dữ liệu được cấu trúc theo một định dạng nhất định và xây dựng lại thành một object — OWASP.

## Tóm tắt

* [Nhận diện Deserialization](#nhận-diện-deserialization)
* [POP Gadgets](#pop-gadgets)
* [Labs](#labs)
* [Tài liệu tham khảo](#tài-liệu-tham-khảo)

## Nhận diện Deserialization

Kiểm tra các phần sau, được trình bày trong những chapter khác:

* [Java deserialization : ysoserial, ...](Java.md)
* [PHP (Object injection) : phpggc, ...](PHP.md)
* [Ruby : universal rce gadget, ...](Ruby.md)
* [Python : pickle, PyYAML, ...](Python.md)
* [.NET : ysoserial.net, ...](DotNET.md)

| Loại Object     | Header (Hex)               | Header (Base64) | Dấu hiệu nhận diện                                                        |
| --------------- | -------------------------- | --------------- | ------------------------------------------------------------------------- |
| .NET ViewState  | `FF 01`                    | `/w`            | Thường được tìm thấy bên trong các hidden input nằm xung quanh HTML form. |
| BinaryFormatter | `0001 0000 00FF FFFF FF01` | `AAEAAAD`       | Base64 decode và kiểm tra chuỗi `FF FF FF FF` dài.                        |
| Java Serialized | `AC ED`                    | `rO`            | Base64 decode và kiểm tra các byte đầu tiên.                              |
| PHP Serialized  | `4F 3A`                    | `Tz`            | Các prefix như `O:, a:, s:, i:, b:` và các length indicator.              |
| Python Pickle   | `80 04 95`                 | `gASV`          | Text: các opcode như `(lp0, S'Test'`.                                     |
| Ruby Marshal    | `04 08`                    | `BAgK`          | Base64 decode và tìm `\x04\x08` ở phần đầu.                               |

## POP Gadgets

> Một POP (Property Oriented Programming) gadget là một đoạn code được triển khai bởi class của ứng dụng, có thể được gọi trong quá trình deserialization.

Đặc điểm của POP gadget:

* Có thể được serialize.
* Có các property public/có thể truy cập.
* Triển khai các method dễ bị tổn thương cụ thể.
* Có quyền truy cập tới các class khác có thể `"callable"`.

## Labs

* [PortSwigger - Modifying serialized objects](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-objects)
* [PortSwigger - Modifying serialized data types](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-modifying-serialized-data-types)
* [PortSwigger - Using application functionality to exploit insecure deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-application-functionality-to-exploit-insecure-deserialization)
* [PortSwigger - Arbitrary object injection in PHP](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-arbitrary-object-injection-in-php)
* [PortSwigger - Exploiting Java deserialization with Apache Commons](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-java-deserialization-with-apache-commons)
* [PortSwigger - Exploiting PHP deserialization with a pre-built gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-php-deserialization-with-a-pre-built-gadget-chain)
* [PortSwigger - Exploiting Ruby deserialization using a documented gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-exploiting-ruby-deserialization-using-a-documented-gadget-chain)
* [PortSwigger - Developing a custom gadget chain for Java deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-developing-a-custom-gadget-chain-for-java-deserialization)
* [PortSwigger - Developing a custom gadget chain for PHP deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-developing-a-custom-gadget-chain-for-php-deserialization)
* [PortSwigger - Using PHAR deserialization to deploy a custom gadget chain](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-using-phar-deserialization-to-deploy-a-custom-gadget-chain)
* [NickstaDB - DeserLab](https://github.com/NickstaDB/DeserLab)

## Tài liệu tham khảo

* [ExploitDB Introduction - Abdelazim Mohammed(@intx0x80) - May 27, 2018](https://web.archive.org/web/20180527082635/https://www.exploit-db.com/docs/english/44756-deserialization-vulnerability.pdf)
* [Exploiting insecure deserialization vulnerabilities - PortSwigger - July 25, 2020](https://web.archive.org/web/20200725143552/https://portswigger.net/web-security/deserialization/exploiting)
* [Instagram's Million Dollar Bug - Wesley Wineberg - December 17, 2015](https://web.archive.org/web/20151217194413/http://exfiltrated.com/research-Instagram-RCE.php)
