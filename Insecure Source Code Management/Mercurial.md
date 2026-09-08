# Mercurial

> Mercurial (còn được gọi là `hg`, bắt nguồn từ ký hiệu hóa học của thủy ngân) là một hệ thống quản lý phiên bản phân tán (DVCS) được thiết kế nhằm mang lại hiệu năng và khả năng mở rộng cao. Được phát triển bởi Matt Mackall và phát hành lần đầu vào năm 2005, Mercurial nổi tiếng với tốc độ, tính đơn giản và khả năng xử lý các codebase lớn.

## Tóm tắt

* [Công cụ](#tools)

  * [rip-hg.pl](#rip-hgpl)
* [Tài liệu tham khảo](#references)

## Công cụ

### rip-hg.pl

* [kost/dvcs-ripper/master/rip-hg.pl](https://raw.githubusercontent.com/kost/dvcs-ripper/master/rip-hg.pl) - Trích xuất các hệ thống quản lý phiên bản phân tán có thể truy cập qua web: SVN/GIT/HG...

  ```powershell
  docker run --rm -it -v /path/to/host/work:/work:rw k0st/alpine-dvcs-ripper rip-hg.pl -v -u
  ```

## Tài liệu tham khảo

* [my-chemical-romance - siunam - February 13, 2023](https://web.archive.org/web/20250712102012/https://siunam321.github.io/ctf/LA-CTF-2023/Web/my-chemical-romance/)
