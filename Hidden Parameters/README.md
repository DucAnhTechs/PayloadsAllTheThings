# Tham số ẩn HTTP
> Các ứng dụng web thường có các tham số ẩn hoặc không được tài liệu hóa mà không được hiển thị trong giao diện người dùng. Fuzzing có thể giúp phát hiện các tham số này, và chúng có thể dễ bị tấn công theo nhiều cách khác nhau.
## Tóm tắt
* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [Dò tìm tham số bằng Bruteforce](#bruteforce-parameters)
    * [Các tham số cũ](#old-parameters)
* [Tài liệu tham khảo](#references)
## Công cụ
* [PortSwigger/param-miner](https://github.com/PortSwigger/param-miner) - Extension Burp để xác định các tham số ẩn, chưa được liên kết.
* [s0md3v/Arjun](https://github.com/s0md3v/Arjun) - Bộ công cụ phát hiện tham số HTTP
* [Sh1Yo/x8](https://github.com/Sh1Yo/x8) - Bộ công cụ phát hiện tham số ẩn
* [tomnomnom/waybackurls](https://github.com/tomnomnom/waybackurls) - Lấy tất cả các URL mà Wayback Machine biết đến cho một domain
* [devanshbatham/ParamSpider](https://github.com/devanshbatham/ParamSpider) - Khai thác URL từ các góc khuất của Web Archives để phục vụ bug hunting/fuzzing/dò tìm thêm
## Phương pháp
### Dò tìm tham số bằng Bruteforce
* Sử dụng wordlist chứa các tham số phổ biến và gửi chúng đi, quan sát các hành vi bất thường từ backend.
    ```ps1
    x8 -u "https://example.com/" -w <wordlist>
    x8 -u "https://example.com/" -X POST -w <wordlist>
    ```
Ví dụ về wordlist:
* [Arjun/large.txt](https://github.com/s0md3v/Arjun/blob/master/arjun/db/large.txt)
* [Arjun/medium.txt](https://github.com/s0md3v/Arjun/blob/master/arjun/db/medium.txt)
* [Arjun/small.txt](https://github.com/s0md3v/Arjun/blob/master/arjun/db/small.txt)
* [samlists/sam-cc-parameters-lowercase-all.txt](https://github.com/the-xentropy/samlists/blob/main/sam-cc-parameters-lowercase-all.txt)
* [samlists/sam-cc-parameters-mixedcase-all.txt](https://github.com/the-xentropy/samlists/blob/main/sam-cc-parameters-mixedcase-all.txt)
### Các tham số cũ
Khám phá tất cả các URL từ mục tiêu của bạn để tìm các tham số cũ.
* Duyệt qua [Wayback Machine](http://web.archive.org/)
* Xem qua các file JS để phát hiện các tham số không còn được sử dụng
## Tài liệu tham khảo
* [Hacker tools: Arjun – The parameter discovery tool - Intigriti - May 17, 2021](https://web.archive.org/web/20230930093635/https://blog.intigriti.com/2021/05/17/hacker-tools-arjun-the-parameter-discovery-tool/)
* [Parameter Discovery: A quick guide to start - YesWeHack - April 20, 2022](http://web.archive.org/web/20220420123306/https://blog.yeswehack.com/yeswerhackers/parameter-discovery-quick-guide-to-start)
