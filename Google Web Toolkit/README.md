# Google Web Toolkit
> Google Web Toolkit (GWT), còn được gọi là GWT Web Toolkit, là một bộ công cụ mã nguồn mở cho phép các nhà phát triển web tạo và duy trì các ứng dụng JavaScript front-end bằng Java. Nó ban đầu được phát triển bởi Google và có bản phát hành đầu tiên vào ngày 16 tháng 5 năm 2006.
## Tóm tắt
* [Công cụ](#tools)
* [Phương pháp](#methodology)
* [Tài liệu tham khảo](#references)
## Công cụ
* [FSecureLABS/GWTMap](https://github.com/FSecureLABS/GWTMap) - GWTMap là một công cụ giúp lập bản đồ bề mặt tấn công của các ứng dụng dựa trên Google Web Toolkit (GWT).
* [GDSSecurity/GWT-Penetration-Testing-Toolset](https://github.com/GDSSecurity/GWT-Penetration-Testing-Toolset) - Một bộ công cụ được tạo ra để hỗ trợ kiểm thử xâm nhập các ứng dụng GWT.
## Phương pháp
* Liệt kê các phương thức của một ứng dụng từ xa thông qua tệp bootstrap của nó và tạo một bản sao lưu cục bộ của mã (chọn permutation một cách ngẫu nhiên):
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --backup
    ```
* Liệt kê các phương thức của một ứng dụng từ xa thông qua một permutation mã cụ thể
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/C39AB19B83398A76A21E0CD04EC9B14C.cache.js
    ```
* Liệt kê các phương thức trong khi định tuyến lưu lượng truy cập thông qua một HTTP proxy:
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --backup -p http://127.0.0.1:8080
    ```
* Liệt kê các phương thức của một bản sao cục bộ (một tệp) của bất kỳ permutation nào cho trước:
    ```ps1
    ./gwtmap.py -F test_data/olympian/C39AB19B83398A76A21E0CD04EC9B14C.cache.js
    ```
* Lọc đầu ra theo một service hoặc phương thức cụ thể:
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --filter AuthenticationService.login
    ```
* Tạo các payload RPC cho tất cả các phương thức của service đã lọc, với đầu ra có màu sắc
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --filter AuthenticationService --rpc --color
    ```
* Tự động kiểm tra (probe) request RPC đã tạo cho phương thức service đã lọc
    ```ps1
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --filter AuthenticationService.login --rpc --probe
    ./gwtmap.py -u http://10.10.10.10/olympian/olympian.nocache.js --filter TestService.testDetails --rpc --probe
    ```
## Tài liệu tham khảo
* [From Serialized to Shell :: Exploiting Google Web Toolkit with EL Injection - Stevent Seeley - May 22, 2017](https://web.archive.org/web/20260220100658/https://srcincite.io/blog/2017/05/22/from-serialized-to-shell-auditing-google-web-toolkit-with-el-injection.html)
* [Hacking a Google Web Toolkit application - thehackerish - April 22, 2021](https://web.archive.org/web/20210227222455/https://thehackerish.com/hacking-a-google-web-toolkit-application/)
