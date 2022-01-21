---
title: "Ruby：JWT 的原理及实现"
author: "Chenghao Zheng"
tags: ["Ruby"]
categories: ["Study notes"]
date: 2022-10-23T03:19:47+01:00
draft: false
---

本篇结合 [JWT Handbook](https://auth0.com/resources/ebooks/jwt-handbook) 讲解针对 [无状态的 HTTP 协议](https://en.wikipedia.org/wiki/Stateless_protocol) 目前主流的一种用户认证方式 Token ，并使用 Ruby 动手实现 JWT 机制来维持用户的登录状态。

# JSON Web Token

JSON Web Token 是一个在有限空间中安全传递凭证的事实标准。JWT 主要应用于登录系统，包括但不限于鉴权、验权、联合身份和客户端 session 等，其由三个部分组成：header、payload 和签名/加密数据，前两部分都是特定结构的 JSON 对象，第三部分依赖于签名/加密的算法（可以在 [jwt.io](https://jwt.io/) 中编辑各部分体会 JWT 的变化）

![](/images/ruby-jwt-example.png)

Header 中的 `alg` 声明了签名/加密算法类型，`typ` 为当前 token 类型；payload 中保存了用户信息，官方定义了七个 payload 中的属性，但它们都不是强制使用的，但需要在使用时避免命名冲突

```ruby
iss: 签发人
sub: 主题
aud: 受众
exp: 过期时间
nbf: 生效时间
iat: 签发时间
jti: 编号
```

至此，一个不安全的 JWT 就可以被构造出来了（实践中很少使用）：

```ruby
def to_base_64
            
end

def encode_pwt_header_and_payload(header: , payload: )

end
```



### JSON Web Signatures

### JSON Web Encryption



# JWT 在工程中的应用





在 [joepie91](http://cryto.net/~joepie91/) 的 [Stop using JWT for sessions](http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/) 中，