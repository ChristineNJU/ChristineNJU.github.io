---
layout: post
title:  "REST API  学习笔记"
categories: Tech
url: "/REST"
---
为了在适应多平台前端的同时，使得开发便捷快速，后端需要为不同的前端提供统一的数据接口，而接口的制定也要符合一定的规范。最近看了 RESTFUL API 相关内容，做了一些笔记。

## REST 是基于资源的
1. 各个资源虽然可能有关联，但依旧是能够简单地切掉这些关联导致相互独立的，所以不会有非常乱的耦合性
2. 对资源的操作就这么几种，所以很容易设计一致的URL
3. 我们明白对资源的读操作是无副作用的，所以能玩缓存

## 区分不同 Methods 的应用场景
HTTP协议提供了很多methods来操作数据：
* GET: 获取某个资源，GET操作应该是幂等（idempotence）的，且无副作用。
* POST: 创建一个新的资源。
* PUT: 替换某个已有的资源。PUT操作虽然有副作用，但其应该是幂等的。
* PATCH（RFC5789）: 修改某个已有的资源（部分更新）。
* DELETE：删除某个资源。DELETE操作有副作用，但也是幂等的。

例如：
* GET /tickets - 获取 tickets 列表
* GET /tickets/12 - 获取一个单独的 ticket
* POST /tickets - 创建一个新的 ticket
* PUT /tickets/12 - 更新 ticket #12
* PATCH /tickets/12 - 部分更新 ticket #12
* DELETE /tickets/12 - 删除 ticket #12

## Headers 规范
1.安全性

OAuth：‘Authorization:token’

HMAC Auth：
‘Authorization:access-key:access-secret’和‘Data:’
access-key和access-secret:前者可以暴露在网络中，后者必须安全保存。当客户端调用API时，用自己的access-secret按照要求对request的headers/body计算HMAC，然后把自己的access-key和HMAC填入Authorization头中。服务器拿到这个头，从数据库（或者缓存）中取出access-key对应的secret，按照相同的方式计算HMAC，如果其与Authorization header中的一致，则请求是合法的，且未被修改过的；否则不合法。

&nbsp;

2.完整性

if-Match:Etag

Etag可以认为是某个资源的一个唯一的版本号。当客户端请求某个资源时，该资源的Etag一同被返回，而当客户端需要修改该资源时，需要通过"If-Match"头来提供这个Etag。服务器检查客户端提供的Etag是否和服务器同一资源的Etag相同，如果相同，才进行修改，否则返回412 precondition failed。

使用Etag可以防止错误更新。比如A拿到了Resource X的Etag X1，B也拿到了Resource X的Etag X1。B对X做了修改，修改后系统生成的新的Etag是X2。这时A也想更新X，由于A持有旧的Etag，服务器拒绝更新，直至A重新获取了X后才能正常更新。

&nbsp;

3.返回数据的格式

Accept：
    服务器需要返回什么样的content。
    如果客户端要求返回"application/xml"，服务器端只能返回"application/json"，那么最好返回status code 406 not acceptable（RFC2616），当然，返回application/json也并不违背RFC的定义。一个合格的REST API需要根据Accept头来灵活返回合适的数据。

If-Modified-Since/If-None-Match：
    如果客户端提供某个条件，那么当这条件满足时，才返回数据，否则返回304 not modified。比如客户端已经缓存了某个数据，它只是想看看有没有新的数据时，会用这两个header之一，服务器如果不理不睬，依旧做足全套功课，返回200 ok，那就既不专业，也不高效了。

If-Match：
    在对某个资源做PUT/PATCH/DELETE操作时，服务器应该要求客户端提供If-Match头，只有客户端提供的Etag与服务器对应资源的Etag一致，才进行操作，否则返回412 precondition failed。这个头非常重要，下文详解。


## 请求数据验证
Request headers是否合法
    比如说你的API需要某个特殊的私有头（e.g. X-Request-ID），那么凡是没有这个头的请求一律拒绝。这可以防止各类漫无目的的webot或crawler的请求，节省服务器的开销。

Request URI和Request body是否合法
    比如说，API只允许querystring中含有query，那么"?sort=desc"这样的请求需要直接被    拒绝。有不少攻击会在querystring和request body里做文章，最好的对应策略是，过滤所有含有不该出现的数据的请求。


## status code 规范
1. 200 (OK) – General success status code. Most common code to indicate success.
2. 201 (CREATED) – Successful creation occurred (via either POST or PUT). Set the Location header to contain a link to the newly-created resource. Response body content may or may not be present.
3. 204 (NO CONTENT) – Status when wrapped responses are not used and nothing is in the body (e.g. DELETE).
4. 304 (NOT MODIFIED) – Used in response to conditional GET calls to reduce band-width usage. If used, must set the Date, Content-Location, Etag headers to what they would have been on a regular GET call. There must be no response body.
5. 400 (BAD REQUEST) – General error when fulfilling the request would cause an invalid state. Domain validation errors, missing data, etc. are some examples.
6. 401 (UNAUTHORIZED) – Error code for a missing or invalid authentication token.
7. 403 (FORBIDDEN) – Error code for user not authorized to perform the operation, doesn't have rights to access the resource, or the resource is unavailable for some reason (e.g. time constraints, etc.).
8. 404 (NOT FOUND) – Used when the requested resource is not found, whether it doesn't exist or if there was a 401 or 403 that, for security reasons, the service wants to mask.
9. 409 (CONFLICT) – Whenever a resource conflict would be caused by fulfilling the request. Duplicate entries, deleting root objects when cascade-delete not supported are a couple of examples.
10.  500 (INTERNAL SERVER ERROR) – The general catch-all error when the server-side throws an exception.

## 要说自己会REST，我觉得至少回答2个问题：
1.对于用户登录和用户退出这两个业务需求，REST指导下的架构和设计如何满足

登入/登出对应的服务端资源应该是session，所以相关api应该如下：
GET /session # 获取会话信息
POST /session # 创建新的会话（登入）
PUT /session # 更新会话信息
DELETE /session # 销毁当前会话（登出）

而注册对应的资源是user，api如下：
GET /user/:id # 获取id用户的信息
POST /user # 创建新的用户（注册）
PUT /user/:id # 更新id用户的信息
DELETE /user/:id # 删除id用户（注销）

2.批量的删除、修改、新增如何满足
使用可以自由添加 body 的库，使用的库不可以自由添加 body 时：
DELETE /posts?id=1,2,3,4
DELETE /posts?id>3&id<5
修改和新增不是直接在 body 里面加 list 就好了嘛？

