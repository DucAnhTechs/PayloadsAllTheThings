# Dependency Confusion (Nhầm Lẫn Phụ Thuộc)
> Tấn công nhầm lẫn phụ thuộc (dependency confusion) hay tấn công thay thế chuỗi cung ứng (supply chain substitution) xảy ra khi một script cài đặt phần mềm bị đánh lừa để lấy về một tệp mã độc từ một kho lưu trữ công khai thay vì tệp có cùng tên dự định từ kho lưu trữ nội bộ.
## Tóm tắt
* [Công cụ](#tools)
* [Phương pháp](#methodology)
    * [Ví dụ NPM](#npm-example)
* [Tài liệu tham khảo](#references)
## Công cụ
* [visma-prodsec/confused](https://github.com/visma-prodsec/confused) - Công cụ kiểm tra các lỗ hổng nhầm lẫn phụ thuộc trong nhiều hệ thống quản lý gói khác nhau
* [synacktiv/DepFuzzer](https://github.com/synacktiv/DepFuzzer) - Công cụ dùng để tìm nhầm lẫn phụ thuộc hoặc dự án mà email của chủ sở hữu có thể bị chiếm đoạt.
## Phương pháp
Tìm các gói `npm`, `pip`, `gem`, phương pháp thực hiện là như nhau: bạn đăng ký một gói công khai có cùng tên với gói riêng tư đang được công ty sử dụng, sau đó chờ đợi nó được sử dụng.
* **DockerHub**: Dockerfile image
* **JavaScript** (npm): package.json
* **MVN** (maven): pom.xml
* **PHP** (composer): composer.json
* **Python** (pypi): requirements.txt
### Ví dụ NPM
* Liệt kê tất cả các gói (ví dụ: package.json, composer.json, ...)
* Tìm gói bị thiếu trên [www.npmjs.com](https://www.npmjs.com/)
* Đăng ký và tạo một gói **công khai** với cùng tên đó
    * Ví dụ về gói: [0xsapra/dependency-confusion-expoit](https://github.com/0xsapra/dependency-confusion-expoit)
## Tài liệu tham khảo
* [Exploiting Dependency Confusion - Aman Sapra (0xsapra) - July 2, 2021](https://web.archive.org/web/20251107024922/https://0xsapra.github.io/website/Exploiting-Dependency-Confusion)
* [Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies - Alex Birsan - February 9, 2021](https://web.archive.org/web/20210209181139/https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)
* [3 Ways to Mitigate Risk When Using Private Package Feeds - Microsoft - March 29, 2021](https://web.archive.org/web/20210210121930/https://azure.microsoft.com/en-gb/resources/3-ways-to-mitigate-risk-using-private-package-feeds/)
* [$130,000+ Learn New Hacking Technique in 2021 - Dependency Confusion - Bug Bounty Reports Explained - February 22, 2021](https://web.archive.org/web/20210223060107/https://www.youtube.com/watch?v=zFHJwehpBrU)
