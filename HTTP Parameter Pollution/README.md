# HTTP Parameter Pollution

> HTTP Parameter Pollution (HPP) là một kỹ thuật né tránh tấn công web cho phép kẻ tấn công tạo ra một request HTTP để thao túng logic web hoặc lấy thông tin ẩn. Kỹ thuật né tránh này dựa trên việc chia một vector tấn công giữa nhiều thực thể của một tham số có cùng tên (?param1=value&param1=value). Vì không có cách chính thức nào để phân tích cú pháp các tham số HTTP, mỗi công nghệ web riêng lẻ có cách phân tích cú pháp và đọc các tham số URL có cùng tên theo cách riêng của nó. Một số lấy lần xuất hiện đầu tiên, một số lấy lần xuất hiện cuối cùng, và một số đọc nó như một mảng. Hành vi này bị kẻ tấn công lợi dụng để vượt qua các cơ chế bảo mật dựa trên mẫu (pattern-based).

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [Bảng Parameter Pollution](#parameter-pollution-table)
    * [Các Payload Parameter Pollution](#parameter-pollution-payloads)
* [Tài liệu tham khảo](#references)

## Công cụ

* **Burp Suite**: Chỉnh sửa request thủ công để kiểm tra các tham số trùng lặp.
* **OWASP ZAP**: Chặn và thao túng các tham số HTTP.

## Phương pháp

HTTP Parameter Pollution (HPP) là một lỗ hổng bảo mật web trong đó kẻ tấn công chèn nhiều thực thể của cùng một tham số HTTP vào một request. Hành vi của server khi xử lý các tham số trùng lặp có thể khác nhau, có khả năng dẫn đến hành vi bất ngờ hoặc có thể bị khai thác.

HPP có thể nhắm mục tiêu vào hai cấp độ:

* Client-Side HPP: Khai thác mã JavaScript đang chạy trên client (trình duyệt).
* Server-Side HPP: Khai thác cách server xử lý nhiều tham số có cùng tên.

**Ví dụ**:

```ps1
/app?debug=false&debug=true
/transfer?amount=1&amount=5000
```

### Bảng Parameter Pollution

Khi ?par1=a&par1=b

| Công nghệ                                      | Kết quả phân tích cú pháp           | kết quả (par1=) |
| ----------------------------------------------- | ------------------------ | --------------- |
| ASP.NET/IIS                                     | Tất cả các lần xuất hiện          | a,b             |
| ASP/IIS                                         | Tất cả các lần xuất hiện          | a,b             |
| Golang net/http - `r.URL.Query().Get("param")`  | Lần xuất hiện đầu tiên         | a               |
| Golang net/http - `r.URL.Query()["param"]`      | Tất cả các lần xuất hiện trong mảng | ['a','b']       |
| IBM HTTP Server                                 | Lần xuất hiện đầu tiên         | a               |
| IBM Lotus Domino                                | Lần xuất hiện đầu tiên         | a               |
| JSP,Servlet/Tomcat                              | Lần xuất hiện đầu tiên         | a               |
| mod_wsgi (Python)/Apache                        | Lần xuất hiện đầu tiên         | a               |
| Nodejs                                          | Tất cả các lần xuất hiện          | a,b             |
| Perl CGI/Apache                                 | Lần xuất hiện đầu tiên         | a               |
| Perl CGI/Apache                                 | Lần xuất hiện đầu tiên         | a               |
| PHP/Apache                                      | Lần xuất hiện cuối cùng          | b               |
| PHP/Zues                                        | Lần xuất hiện cuối cùng          | b               |
| Python Django                                   | Lần xuất hiện cuối cùng          | b               |
| Python Flask                                    | Lần xuất hiện đầu tiên         | a               |
| Python/Zope                                     | Tất cả các lần xuất hiện trong mảng | ['a','b']       |
| Ruby on Rails                                   | Lần xuất hiện cuối cùng          | b               |

### Các Payload Parameter Pollution

* Tham số trùng lặp:

    ```ps1
    param=value1&param=value2
    ```

* Chèn mảng (Array Injection):

    ```ps1
    param[]=value1
    param[]=value1&param[]=value2
    param[]=value1&param=value2
    param=value1&param[]=value2
    ```

* Chèn dạng mã hóa (Encoded Injection):

    ```ps1
    param=value1%26other=value2
    ```

* Chèn lồng nhau (Nested Injection):

    ```ps1
    param[key1]=value1&param[key2]=value2
    ```

* Chèn JSON (JSON Injection):

    ```ps1
    {
        "test": "user",
        "test": "admin"
    }
    ```

## Tài liệu tham khảo

* [How to Detect HTTP Parameter Pollution Attacks - Acunetix - January 9, 2024](https://web.archive.org/web/20260112091623/https://www.acunetix.com/blog/whitepaper-http-parameter-pollution/)
* [HTTP Parameter Pollution - Itamar Verta - December 20, 2023](https://web.archive.org/web/20190721110154/https://www.imperva.com/learn/application-security/http-parameter-pollution/)
* [HTTP Parameter Pollution in 11 minutes - PwnFunction - January 28, 2019](https://web.archive.org/web/20190212095035/https://www.youtube.com/watch?v=QVZBl8yxVX0)
