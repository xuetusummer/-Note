- [GraphQL 安全](#graphql-安全)
  - [基础查询](#基础查询)
    - [GraphQL Field Suggestions](#graphql-field-suggestions)
    - [Interospection Query(自省查询)](#interospection-query自省查询)
  - [Json --> form-urlencoded --> CSRF](#json----form-urlencoded----csrf)
# GraphQL 安全
## 基础查询
### GraphQL Field Suggestions
GraphQL字段自动推测,当开启开功能时，如果输入不存在的字段,GraphQL会自动回显相似的字段。
![](2022-10-21-17-46-55.png)
### Interospection Query(自省查询)
该查询可以返回所有系统定义的字段。
```graphql
  query IntrospectionQuery {
    __schema {
      queryType { name }
      mutationType { name }
      subscriptionType { name }
      types {
        ...FullType
      }
      directives {
        name
        description
        args {
          ...InputValue
        }
        onOperation
        onFragment
        onField
      }
    }
  }

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
    type { ...TypeRef }
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
        }
      }
    }
  }
```
```http
POST /graphql HTTP/2
Host: test.com
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:55.0) Gecko/20100101 Firefox/55.0
Content-Length: 746
Content-Type: application/json

{"query": "query IntrospectionQuery{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}"}
```
## Json --> form-urlencoded --> CSRF

正常GraphQL查询的content-type为application/json
```
POST /graphql HTTP/1.1
Host: redacted
Connection: close
Content-Length: 100
accept: */*
User-Agent: ...
content-type: application/json
Referer: https://redacted/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: ...

{"operationName":null,"variables":{},"query":"{\n  user {\n    firstName\n    __typename\n  }\n}\n"}
```
而由于中间件转换的原因,有时候服务器也会接受同样以form-urlencoded格式的post请求
```
POST /graphql HTTP/1.1
Host: redacted
Connection: close
Content-Length: 72
accept: */*
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 11_2_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.82 Safari/537.36
Content-Type: application/x-www-form-urlencoded
Referer: https://redacted
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: ...

query=%7B%0A++user+%7B%0A++++firstName%0A++++__typename%0A++%7D%0A%7D%0A
```
有时候该post请求则可能会发生csrf漏洞,因为开发者认为GraphQL的格式为Json格式,无法从表单创建,没有办法从urlencoded请求到json,而对GraphQL请求放弃做CSRF的检查
