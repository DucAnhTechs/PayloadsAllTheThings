# Rò rỉ API Key và Token

> API key và token là các hình thức xác thực thường được dùng để quản lý quyền truy cập vào các dịch vụ công khai và riêng tư. Việc để rò rỉ những dữ liệu nhạy cảm này có thể dẫn đến truy cập trái phép, làm suy yếu bảo mật, và có nguy cơ gây ra rò rỉ dữ liệu.

## Mục lục

- [Công cụ](#tools)
- [Phương pháp](#methodology)
    - [Các nguyên nhân phổ biến gây rò rỉ](#common-causes-of-leaks)
    - [Xác thực API Key](#validate-the-api-key)
- [Giảm thiểu bề mặt tấn công](#reducing-the-attack-surface)
- [Tài liệu tham khảo](#references)

## Công cụ

- [aquasecurity/trivy](https://github.com/aquasecurity/trivy) - Công cụ quét lỗ hổng và cấu hình sai đa năng, cũng có khả năng tìm kiếm API key/secret.
- [blacklanternsecurity/badsecrets](https://github.com/blacklanternsecurity/badsecrets) - Thư viện phát hiện các secret đã biết hoặc yếu trên nhiều nền tảng.
- [irsdl/crapsecrets](https://github.com/irsdl/crapsecrets) - Thư viện phát hiện các secret đã biết trên nhiều framework web.
- [d0ge/sign-saboteur](https://github.com/d0ge/sign-saboteur) - SignSaboteur là extension của Burp Suite dùng để chỉnh sửa, ký, xác minh nhiều loại web token đã được ký.
- [mazen160/secrets-patterns-db](https://github.com/mazen160/secrets-patterns-db) - Secrets Patterns DB: Cơ sở dữ liệu mã nguồn mở lớn nhất để phát hiện secret, API key, mật khẩu, token, v.v.
- [momenbasel/KeyFinder](https://github.com/momenbasel/KeyFinder) - Công cụ giúp bạn tìm key trong khi lướt web.
- [streaak/keyhacks](https://github.com/streaak/keyhacks) - Repository cho thấy các cách nhanh chóng để kiểm tra tính hợp lệ của API key bị rò rỉ trong chương trình bug bounty.
- [trufflesecurity/truffleHog](https://github.com/trufflesecurity/truffleHog) - Tìm kiếm credential ở mọi nơi.
- [projectdiscovery/nuclei-templates](https://github.com/projectdiscovery/nuclei-templates) - Sử dụng các template này để kiểm tra một API token trên nhiều endpoint dịch vụ API.

    ```powershell
    nuclei -t token-spray/ -var token=token_list.txt
    ```

## Phương pháp

- **API Key**: Định danh duy nhất dùng để xác thực các request liên quan đến project hoặc ứng dụng của bạn.
- **Token**: Các security token (như OAuth token) cấp quyền truy cập vào tài nguyên được bảo vệ.

### Các nguyên nhân phổ biến gây rò rỉ

- **Hardcode trong mã nguồn**: Lập trình viên có thể vô tình để lại API key hoặc token trực tiếp trong mã nguồn.

    ```py
    # Ví dụ về API key bị hardcode
    api_key = "1234567890abcdef"
    ```

- **Repository công khai**: Vô tình commit các key và token nhạy cảm lên hệ thống quản lý phiên bản công khai như GitHub.

    ```ps1
    ## Quét một tổ chức trên Github
    docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest github --org=trufflesecurity
    
    ## Quét một Repository GitHub, các Issue và Pull Request của nó
    docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest github --repo https://github.com/trufflesecurity/test_keys --issue-comments --pr-comments
    ```

- **Hardcode trong Docker Image**: API key và credential có thể bị hardcode trong các Docker image được lưu trữ trên DockerHub hoặc registry riêng.

    ```ps1
    # Quét một Docker image để tìm secret đã được xác minh
    docker run --rm -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest docker --image trufflesecurity/secrets
    ```

- **Log và thông tin Debug**: Key và token có thể vô tình bị ghi log hoặc in ra trong quá trình debug.

- **File cấu hình**: Bao gồm key và token trong các file cấu hình có thể truy cập công khai (ví dụ: file .env, config.json, settings.py, hoặc .aws/credentials).

### Xác thực API Key

Nếu cần hỗ trợ xác định dịch vụ đã tạo ra token, có thể tham khảo [mazen160/secrets-patterns-db](https://github.com/mazen160/secrets-patterns-db). Đây là cơ sở dữ liệu mã nguồn mở lớn nhất để phát hiện secret, API key, mật khẩu, token, v.v. Cơ sở dữ liệu này chứa các mẫu regex cho nhiều loại secret khác nhau.

```yaml
patterns:
  - pattern:
      name: AWS API Gateway
      regex: '[0-9a-z]+.execute-api.[0-9a-z._-]+.amazonaws.com'
      confidence: low
  - pattern:
      name: AWS API Key
      regex: AKIA[0-9A-Z]{16}
      confidence: high
```

Sử dụng [streaak/keyhacks](https://github.com/streaak/keyhacks) hoặc đọc tài liệu của dịch vụ để tìm cách nhanh chóng xác minh tính hợp lệ của một API key.

- **Ví dụ**: Telegram Bot API Token

    ```ps1
    curl https://api.telegram.org/bot<TOKEN>/getMe
    ```

## Giảm thiểu bề mặt tấn công

Kiểm tra sự tồn tại của private key hoặc AWS credential trước khi commit thay đổi của bạn vào repository GitHub.

Thêm các dòng sau vào file `.pre-commit-config.yaml` của bạn.

```yml
-   repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v3.2.0
    hooks:
    -   id: detect-aws-credentials
    -   id: detect-private-key
```

## Tài liệu tham khảo

- [Finding Hidden API Keys & How to Use Them - Sumit Jain - August 24, 2019](https://web.archive.org/web/20191012175520/https://medium.com/@sumitcfe/finding-hidden-api-keys-how-to-use-them-11b1e5d0f01d)
- [Introducing SignSaboteur: Forge Signed Web Tokens with Ease - Zakhar Fedotkin - May 22, 2024](https://web.archive.org/web/20240522172244/https://portswigger.net/research/introducing-signsaboteur-forge-signed-web-tokens-with-ease)
- [Private API Key Leakage Due to Lack of Access Control - yox - August 8, 2018](https://web.archive.org/web/20211208043535/https://hackerone.com/reports/376060)
- [Saying Goodbye to My Favorite 5 Minute P1 - Allyson O'Malley - January 6, 2020](https://web.archive.org/web/20250714230057/https://www.allysonomalley.com/2020/01/06/saying-goodbye-to-my-favorite-5-minute-p1/)
