+++
title = "HTTP API设计"
tags = ["http", "api"]
date = "2014-08-02"
categories = [
  "HTTP"
]
+++

## HTTP API设计指导
via [github@geemus](https://github.com/interagent/http-api-design)

### 介绍

作者的本意是介绍[heroku](https://heroku.com/ "神奇的平台")平台的 `http-json` API设计 [heroku官方文档](https://devcenter.heroku.com/articles/platform-api-reference)
本文意在推荐一套成熟的api设计模式来减少读者在api设计上遇到的坑.无论如何,这篇指导都比较适合
初涉api设计,或者在api设计上遇到瓶颈的同学.学习之-

### 准备工作

#### 1. 需要tls

强制依赖tls握手协议,所有不通过tls的请求都视为非法的,阻止并返回 `403` 错误.不推荐使用重定向动作,
因为重定向行为本身就接受了所有的非法的客户端请求,而且加倍了服务器流量,重定向的请求同样没有通过tls,这
是个非常差的设计.

#### 2. 在accepts头参数中表明api的版本

不要使用默认的版本,要让客户端请求明确表明它们需要的api版本通过在`accepts`头参数里面加上版本号.
如:
    
    Accept: application/vnd.heroku+json; version=3

#### 3. 使用Etags

使用 `etag` 头参数来缓存数据,客户端只需回传 `If-None-Match` 验证数据是否过期,这样可以节省服务器资源.

#### 4. 通过Request-Ids追踪请求

使用 `Request-Id` 头参数,协助追中和debug请求.

#### 5. Content-Range标记页码

### 请求 `Request`

#### 1. 使用状态码

状态码介绍内容省略,可以通过[google](google.com)以及[baidu](baidu.com)等搜索出详细介绍.[HTTP response code spec](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)

#### 2. 返回完整的资源

请求成功后(200, 201状态码),尽可能的返回完整的资源.即使是 `put/patch` 和 `delete` 请求.
如:

```
$ curl -X DELETE \  
 https://service.com/apps/1f9b/domains/0fd4
```  
```
HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
 "created_at": "2012-01-01T12:00:00Z",
 "hostname": "subdomain.example.com",
 "id": "01234567-89ab-cdef-0123-456789abcdef",
 "updated_at": "2012-01-01T12:00:00Z"
}
```
```
$ curl -X DELETE \  
 https://service.com/apps/1f9b/dynos/05bd
```
```
HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### 3. 接受json参数请求

```
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'
```

```
{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### 4. 使用restful的路径格式

##### 资源名
　　
使用资源名词的复数形式,e.g. `books, apples, users` 但是也存在列外,比如大部分情况下一个 `user` 只有一个 `account` 
e.g.

```  
/users/:user/account/actions/:action/
```

##### 资源操作

单单展示资源的时候,路径名不需要添加动作,堆资源进行操作的时候需要满足如下规则:

```
/resources/:resource/actions/:action
```

e.g.

```
/books/{book_id}/actions/delete
```


#### 5. 路径使用小写字母

使用小写字母和破折号标示路径. e.g.

```
service-api.com/users
service-api.com/app-setups
```
 
json 或者 yaml 字段属性, 使用下划线链接. e.g.

```
service_class: "first"
```

#### 6. 路径中资源可以用非ID标示

前面介绍过,资源路径标示. `/resources/{resource_id}` 但是总会存在,资源标示符未非数字的情况,如: `uuid` 
出现此类情况,可以使用其他标示符填写路径. e.g.

```
$ curl https://service.com/users/{app_id_or_name}
$ curl https://service.com/users/97addcf0-c182
$ curl https://service.com/users/www-prod
```
不要只允许使用IDs标示资源


#### 7. 因尽量减少路径的深层次嵌套

往往会因为数据模型的深层次嵌套导致路径的深层次嵌套. e.g.

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

这种写法导致路径的理解困难,难读.推荐做法

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```
    
路径集合也能标示主从关系.并且易读写.

### 反馈 `Response`

#### 1. 资源的唯一标示ID

为每个资源创建一个唯一ID. e.g. `UUID`
不要使用数据表自增长ID作为标示符.并使用小写字母 `8-4-4-4-12` 格式. e.g.

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### 2. 记录资源创建和更新时间戳

提供 `created_at` 和 `updated_at` e.g.

```
{
  ...
  "created_at": "2014-08-01T12:00:00Z",
  "updated_at": "2014-08-01T13:00:00Z",
  ...
}
```

#### 3. 使用UTC时间和ISo8601格式

e.g.

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### 4. 包含并嵌套外键关系

使用:

```
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  ...
}
```

而非:

```
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  ...
}
```

原因:这样做的目的是为了能够方便的扩展关系资源包含的属性,而不需要修改或引入数据结构

```
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  ...
}
```

#### 5. 错误数据返回格式应该是经过设计的

一般包括错误 `id` 易理解的信息 `message` 以及一个可以获取解决方案的 `url`
e.g.

```
HTTP/1.1 429 Too Many Requests
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

#### 6. 标示请求速率限制状态

使用 `RateLimit-Remaining` 头参数,标示当前请求限制状态

#### 7. 压缩返回的json文件

去掉不必要的换行符,空格.
使用:

```
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z", "created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```
    
替代:
    
```
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

但是这并不以为着精简json的内容,更不能通过添加查询参数( `?pretty=true` ), 或者在 `accept` 头参数中添加属性( `Accept: application/vnd.heroku+json; version=3; indent=4;` )

### 注意



>1. 提供机器识别的json数据结构. [tool prmd](https://github.com/interagent/prmd)
>2. 提供简单明了的API使用文档,即使这有你一个人使用.
>3. 提供可以及时执行的列子. 
>
>>      curl -is https://$TOKEN@service.com/users
>
>4. 加入change log

#### **声明**:转载请注明出处via[xus](https://github.com/JohnSmithX)
