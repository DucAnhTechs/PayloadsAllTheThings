# Lỗi Logic Nghiệp vụ (Business Logic Errors)

> Lỗi logic nghiệp vụ, còn được gọi là lỗ hổng logic nghiệp vụ, là một loại lỗ hổng ứng dụng bắt nguồn từ logic nghiệp vụ của ứng dụng, tức là phần của chương trình xử lý các quy tắc và quy trình kinh doanh thực tế. Các quy tắc này có thể bao gồm các mô hình định giá, giới hạn giao dịch, hoặc trình tự các thao tác cần phải tuân theo trong một quy trình nhiều bước.

## Mục lục

* [Phương pháp](#methodology)
    * [Kiểm thử tính năng Review](#review-feature-testing)
    * [Kiểm thử tính năng mã giảm giá](#discount-code-feature-testing)
    * [Thao túng phí giao hàng](#delivery-fee-manipulation)
    * [Chênh lệch giá tiền tệ (Currency Arbitrage)](#currency-arbitrage)
    * [Khai thác tính năng Premium](#premium-feature-exploitation)
    * [Khai thác tính năng hoàn tiền](#refund-feature-exploitation)
    * [Khai thác giỏ hàng/danh sách yêu thích](#cartwishlist-exploitation)
    * [Kiểm thử bình luận Thread](#thread-comment-testing)
    * [Lỗi làm tròn số (Rounding Error)](#rounding-error)
* [Tài liệu tham khảo](#references)

## Phương pháp

Khác với các loại lỗ hổng bảo mật khác như SQL injection hay cross-site scripting (XSS), lỗi logic nghiệp vụ không dựa vào các vấn đề trong chính mã nguồn (như dữ liệu đầu vào của người dùng không được lọc). Thay vào đó, chúng lợi dụng chức năng bình thường, đúng như thiết kế của ứng dụng, nhưng sử dụng theo những cách mà nhà phát triển không lường trước được và gây ra những hậu quả không mong muốn.

Các ví dụ phổ biến về Lỗi Logic Nghiệp vụ.

### Kiểm thử tính năng Review

* Đánh giá xem bạn có thể đăng một review sản phẩm với vai trò là người mua đã xác minh (verified reviewer) mà không cần đã mua sản phẩm đó hay không.
* Thử cung cấp một xếp hạng (rating) nằm ngoài thang điểm tiêu chuẩn, ví dụ, một số 0, 6 hoặc số âm trong hệ thống thang điểm 1 đến 5.
* Kiểm tra xem cùng một người dùng có thể đăng nhiều rating cho một sản phẩm duy nhất hay không. Điều này hữu ích trong việc phát hiện các race condition tiềm ẩn.
* Xác định xem trường tải file lên (file upload) có cho phép tất cả các phần mở rộng (extension) hay không; các nhà phát triển thường bỏ qua việc bảo vệ các endpoint này.
* Điều tra khả năng đăng review giả mạo người dùng khác.
* Thử tấn công Cross-Site Request Forgery (CSRF) trên tính năng này, vì nó thường không được bảo vệ bởi token.

### Kiểm thử tính năng mã giảm giá

* Thử áp dụng cùng một mã giảm giá nhiều lần để đánh giá xem nó có thể được tái sử dụng hay không.
* Nếu mã giảm giá là duy nhất, hãy đánh giá race condition bằng cách áp dụng cùng một mã cho hai tài khoản cùng một lúc.
* Kiểm tra Mass Assignment hoặc HTTP Parameter Pollution để xem bạn có thể áp dụng nhiều mã giảm giá khi ứng dụng chỉ được thiết kế để chấp nhận một mã hay không.
* Kiểm tra các lỗ hổng do thiếu việc làm sạch (sanitization) dữ liệu đầu vào như XSS, SQL Injection trên tính năng này.
* Thử áp dụng mã giảm giá cho các sản phẩm không được giảm giá bằng cách thao túng request phía server.

### Thao túng phí giao hàng

* Thử nghiệm với các giá trị âm cho phí giao hàng để xem nó có làm giảm số tiền cuối cùng hay không.
* Đánh giá xem giao hàng miễn phí có thể được kích hoạt bằng cách chỉnh sửa tham số hay không.

### Chênh lệch giá tiền tệ (Currency Arbitrage)

* Thử thanh toán bằng một loại tiền tệ, ví dụ USD, và yêu cầu hoàn tiền bằng một loại tiền tệ khác, như EUR. Sự chênh lệch trong tỷ giá chuyển đổi có thể mang lại lợi nhuận.

### Khai thác tính năng Premium

* Khám phá khả năng truy cập vào các phần hoặc endpoint chỉ dành cho tài khoản premium mà không cần có gói đăng ký hợp lệ.
* Mua một tính năng premium, hủy nó, và xem liệu bạn có thể vẫn sử dụng nó sau khi được hoàn tiền hay không.
* Tìm kiếm các giá trị true/false trong request/response dùng để xác thực quyền truy cập premium. Sử dụng các công cụ như Match & Replace của Burp để thay đổi các giá trị này nhằm truy cập premium trái phép.
* Xem xét cookie hoặc local storage để tìm các biến xác thực quyền truy cập premium.

### Khai thác tính năng hoàn tiền

* Mua một sản phẩm, yêu cầu hoàn tiền, và xem sản phẩm đó có còn truy cập được hay không.
* Tìm kiếm các cơ hội cho chênh lệch giá tiền tệ.
* Gửi nhiều yêu cầu hủy cho một gói đăng ký để kiểm tra khả năng được hoàn tiền nhiều lần.

### Khai thác giỏ hàng/danh sách yêu thích

* Kiểm tra hệ thống bằng cách thêm sản phẩm với số lượng âm, cùng với các sản phẩm khác, để cân bằng tổng số tiền.
* Thử thêm số lượng sản phẩm nhiều hơn số lượng có sẵn.
* Kiểm tra xem một sản phẩm trong danh sách yêu thích hoặc giỏ hàng của bạn có thể được chuyển sang giỏ hàng của người dùng khác hoặc bị xóa khỏi đó hay không.

### Kiểm thử bình luận Thread

* Kiểm tra xem có giới hạn về số lượng bình luận trên một thread hay không.
* Nếu một người dùng chỉ có thể bình luận một lần, hãy sử dụng race condition để xem liệu nhiều bình luận có thể được đăng hay không.
* Nếu hệ thống cho phép bình luận bởi những người dùng đã xác minh hoặc có đặc quyền, hãy thử mô phỏng các tham số này và xem liệu bạn có thể bình luận được hay không.
* Thử đăng bình luận giả mạo người dùng khác.

### Lỗi làm tròn số (Rounding Error)

Báo cáo [hackerone #176461](https://web.archive.org/web/20170303191338/https://hackerone.com/reports/176461) mô tả một lỗ hổng logic nghiệp vụ trong một nền tảng tiền điện tử (sử dụng XBT/Bitcoin), nơi kẻ tấn công khai thác một lỗi làm tròn số trong hệ thống chuyển tiền nội bộ để tạo ra tiền từ hư không.

Kẻ tấn công khởi tạo một giao dịch chuyển 0.000000005 XBT (0.5 satoshi), giá trị này thấp hơn độ chính xác tối thiểu của hệ thống là 1 satoshi.

* Số dư của người gửi không thay đổi. Thuật toán có thể đã làm tròn xuống thành 0 satoshi.
* Số dư của người nhận tăng thêm 1 satoshi (0.00000001). Thuật toán có thể đã làm tròn lên thành 1 satoshi.

Kẻ tấn công đã tạo ra 0.00000001 XBT từ hư không, vì không có giới hạn tần suất (rate limit), OTP, hay phát hiện gian lận, kẻ tấn công có thể tự động hóa quy trình này và lặp lại nó vô hạn lần, thực chất là in tiền.

Trong ví dụ này, thay vì làm tròn và từ chối hoặc áp dụng mức chuyển tối thiểu, hệ thống bỏ qua việc trừ tiền từ người gửi và vẫn cộng tiền cho người nhận.

## Tài liệu tham khảo

* [Business Logic Vulnerabilities - PortSwigger - March 5, 2026](https://web.archive.org/web/20260305155804/https://portswigger.net/web-security/logic-flaws)
* [Business Logic Vulnerability - OWASP - April 22, 2020](https://web.archive.org/web/20200422002600/https://owasp.org/www-community/vulnerabilities/Business_logic_vulnerability)
* [CWE-840: Business Logic Errors - CWE - March 24, 2011](https://web.archive.org/web/20260304013031/https://cwe.mitre.org/data/definitions/840.html)
* [Examples of Business Logic Vulnerabilities - PortSwigger - September 22, 2020](https://web.archive.org/web/20200922175829/https://portswigger.net/web-security/logic-flaws/examples)
