# Session管理

## JWT是什么？

**JSON Web Tokens**，简称==JWT==，是一种开放标准（RFC7519）它定义了一种紧凑和自包含的方式，用于在各方之间作为JSON对象安全地传输信息。

一般跨域认证流程：

1. 用户向服务器发送用户名和密码。
2. 服务器验证通过后，在当前对话（session）里面保存相关数据，比如用户角色、登录时间等等
3. 服务器向用户返回一个 session_id，写入用户的 Cookie。
4. 用户随后的每一次请求，都会通过 Cookie，将 session_id 传回服务器。
5. 服务器收到 session_id，找到前期保存的数据，由此得知用户的身份。

> Cookie是一种存储方式，概念类似Local Storage，存在于HTTP头部中。
>
> 用户浏览网站，由服务器创建并由浏览器存放在用户计算机或其他设备的小文本文件。
>
> Cookie使Web服务器能在用户的设备存储状态信息（如添加到在线商店购物车中的商品）或跟踪用户的浏览活动（如点击特定按钮、登录或记录历史）

如果是服务器集群，就需要Session共享，每台服务器都能够读到Session。

## 工作原理

**JWT**

1. 用户登录后，服务器生成JWT（包含用户信息，经过签名加密）。
2. 客户端存储JWT（可以是LocalStorage、SessionStorage或Cookie）。
3. 客户端每次请求时携带JWT，服务器通过解码和验证签名来确认用户身份，无需存储状态。

**Session**

1. 用户登录后，服务器生成Session数据，并返回一个会话ID给客户端。

2. 客户端每次请求时携带Session ID，服务器验证后读取会话数据。

3. Session通常需要服务器维护一个存储机制，比如Redis、内存或数据库。

## Session与JWT的存储内容

### **Session**

* 状态存储在**服务器**上，通常通过内存或数据库保存用户的会话数据。

* 客户端仅保存一个会话ID（通常通过Cookie或URL参数传递），服务器通过该ID检索用户的会话数据。

```json
# 可能保存的几种常见的状态信息
# 1.用户身份信息
# 2.认证状态：登录状态、CSRF Token
# 3.应用特定的业务数据
# 4.安全相关的状态
# 5.状态标志
# 6.会话元信息

# 保存示例
{
  "session_id": "abcd1234",
  "user_id": 1001,
  "role": "admin",
  "is_logged_in": true,
  "csrf_token": "xyz987",
  "cart_items": [
    {"product_id": 101, "quantity": 2},
    {"product_id": 202, "quantity": 1}
  ],
  "preferences": {
    "language": "en",
    "theme": "dark"
  },
  "last_active": "2024-11-26T12:34:56Z",
  "expires_at": "2024-11-26T14:00:00Z"
}

```

### **JWT**

存储方式

* 状态存储在**客户端**，JWT是一个自包含的Token，直接携带用户的信息（通常是加密后的）。

* 无需服务器存储，Token通过Cookie、HTTP Header或其他方式传递。

JWT内容包含三个部分

* Header
* Payload
* Signature

<img src="https://cdn.beekka.com/blogimg/asset/201807/bg2018072304.jpg" alt="img" style="zoom:80%;" />

==Header== 部分是一个 JSON 对象，描述 JWT 的元数据。

```javascript
{
  "alg": "HS256",
  "typ": "JWT"
}
```

上面代码中，`alg`属性表示签名的算法（algorithm），默认是 HMAC SHA256（写成 HS256）；`typ`属性表示这个令牌（token）的类型（type），JWT 令牌统一写为`JWT`。

**==Payload==**部分也是一个 JSON 对象，用来存放实际需要传递的数据。JWT 规定了7个官方字段，供选用。

 - iss (issuer)：签发人
 - exp (expiration time)：过期时间
 - sub (subject)：主题
 - aud (audience)：受众
 - nbf (Not Before)：生效时间
 - iat (Issued At)：签发时间
 - jti (JWT ID)：编号

除了官方字段，你还可以在这个部分定义私有字段，下面就是一个例子。

 ```javascript
 {
   "sub": "1234567890",
  	"name": "John Doe",
   "admin": true
}
 ```

注意，JWT 默认是不加密的，任何人都可以读到，所以不要把秘密信息放在这个部分。

这个 JSON 对象也要使用 Base64URL 算法转成字符串。

==Signature== 部分是对前两部分的签名，防止数据篡改。

首先，需要指定一个密钥（secret）。这个密钥只有服务器才知道，不能泄露给用户。然后，使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。

 ```javascript
 HMACSHA256(
   base64UrlEncode(header) + "." +
   base64UrlEncode(payload),
   secret)
 ```

算出签名以后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分之间用"点"（`.`）分隔，就可以返回给用户。

**特性对照表**

| 特性       | Session        | JWT                 |
| ---------- | -------------- | ------------------- |
| 存储位置   | 服务器         | 客户端              |
| 是否无状态 | 否             | 是                  |
| 扩展性     | 较差           | 较好                |
| 安全性     | 基于Session ID | 基于Token内容和签名 |
| 应用场景   | 传统Web应用    | 分布式，前后端分离  |

## 业务系统中的应用

采用Token-Base方式进行会话管理，利用类“双Token”的方式实现用户的==无感刷新==，保证活跃用户不会因为``Token``过期要求重新登陆。

1. 调用登录接口，``Server``生成``Token``，将``Token``返回至前端。
2. 前端后续的接口请求需携带该``Token``，后端服务校验``Token``中的有效期，若在有效期内，则正常响应该请求；若不在==初始有效期==内，校验是否在==续期==内，若在续期内，将重新生成``Token``，返回给前端。
3. 若不在续期内，需要重新调度登录接口获取新的``Token``。

> Server会指定两个有效期，初始有效期、续期。初始有效期生成``Token``时，存入``Token``中，由``Token``类库校验。``Redis``存储创建``Token``时的时间戳，需要校验时，从``Redis``中读取时间与续期进行校验。

Redis Token信息数据格式

```Json
{
    "token":{
        "token":string,
        "time":unsigned long long,
    }
}
```

## 参考

[JSON Web Token (JWT) - IBM Documentation](https://www.ibm.com/docs/en/cics-ts/6.x?topic=cics-json-web-token-jwt)
