---
title: Server 端的认证——拥抱 JWT（一）
date: 2016-11-04 18:06:56
tags: [jwt]
---

#### What is Json Web Token

根据官网的定义，JWT 是一套开放的标准（RFC 7519），它定义了一套简洁的（compact）、自包含的（self-contained）方案，来让我们安全地在客户端和服务器之间传递 JSON 格式的信息。

#### Advantages

- 体积小，因而传输速度快
- 传输方式多样，可以通过 URL/POST 参数/HTTP 头部 等方式传输
- 严谨的结构化。它自身（在 payload 中）就包含了所有与用户相关的验证消息，如用户可访问路由、访问有效期等信息，服务器无需再去连接数据库验证信息的有效性，并且 payload 支持为你的应用而定制化
- 支持跨域验证，多应用于单点登录。

> 单点登录（Single Sign On）：在多个应用系统中，用户只需登陆一次，就可以访问所有相互信任的应用。

#### WHY JWT

除了上面说到的优点之外，相比传统的服务端验证， JWT 还有以下优点。

- 充分依赖无状态 API ，契合 RESTful 设计原则
- 易于实现 CDN，将静态资源分布式管理
- 验证解耦，无需使用特定的身份验证方案， token 可以在任何地方生成
- 比 cookie 更支持原生移动端应用

1. 关于状态  
首先我们先看一下，什么是状态，什么是有状态与无状态。

> 状态：请求的状态是 client 与 server 交互过程中，保存下来的相关信息，客户端的保存在 page/request/session/application 或者全局作用域中，而 server 的一般存在 session 中。

> 有状态 API：server 保存了 client 的请求状态， server 会通过 client 传递的 sessionID 在其 session 作用域内找到之前交互的信息并应答。

> 无状态 API：无状态是 RESTful 架构设计的一个非常主要的原则。无状态 API 的每一个请求都是独立的，它要求由客户端保存所有需要的认证信息，每次发请求都要带上自己的状态，以 url 的形式提交包含了 cookies 等状态的数据。

JWT 就很好地体现了无状态原则。用户登陆之后，服务器会返回给他一个 token，由他保存在本地，在这之后的对服务器的访问都要带上这串 JWT ,来获得访问服务器相关路由、服务及资源的权限。比如单点登录就比较多地使用了 JWT，因为它的体积小，并且简单处理（使用 HTTP 头带上 Bearer 属性 + token ）就可以支持跨域操作。

2. 分布式管理  
在传统的 session 验证中，服务端必须保存 session ID，用于与用户传过来的 cookie 验证。而在一开始保存 session ID 时， 只会保存在一台服务器上，所以只能由一个 server 应答，就算其他服务器有空闲也无法应答，因此也利用不到分布式服务器的优点。  
而 JWT 依赖的是在客户端本地保存验证信息，不需要利用服务器保存的信息来验证，所以任意一台服务器都可以应答，服务器的资源也被较好地利用。

3. 验证解耦  
只要拥有生成 token 所需的验证信息，在何处都可以调用 token 生成接口，无需繁琐的耦合的验证操作，可谓是一次生成，永久使用。

4. 对原生应用的支持（我对移动端开发不够深入，这点不是很清楚） 
原生的移动应用对 cookie 与 session 的支持不够好，而对 token 的方式支持较好。

除此之外，JWT 的可靠的结构化的标准，也是我们选择它的一大原因。特别是使用 nodejs 开发时，嗯，node 大法好！


#### client 使用 JWT 与 server 交互的过程

![image](http://ac-Myg6wSTV.clouddn.com/055d6a84b0aef29266e5.png)

首先，拥有某网站账号的某 client 使用自己的账号密码发送 post 请求 login，由于这是首次接触，server 会校验账号与密码是否合法，如果一致，则根据密钥生成一个 token 并返回，client 收到这个 token 并保存在本地的 localStorage。在这之后，需要访问一个受保护的路由或资源时，而只要附加上你保存在本地的 token（通常使用 Bearer 属性放在 Header 的 Authorization 属性中），server 会检查这个 token 是否仍有效，以及其中的校验信息是否正确，再做出相应的响应。

#### JWT 由三部分组成：Header/Payload/Signature

```
// Header
{
  "alg": "HS256",
  "type": "JWT"
}

// Payload
{
  // reserved claims
  "iss": "a.com",
  "exp": "1d",
  // public claims
  "http://a.com": true,
  // private claims
  "company": "A",
  "awesome": true
}

// $Signature
HS256(Base64(Header) + "." + Base64(Payload), secretKey)


JWT = {Base64(Header), Base64(Payload), $Signature}
```

第一部分是 Header。首先声明一个 JSON 对象，对象里有一个 type 属性，值为 JWT ，以及 alg 属性，值为 HS256，表明最终使用的加密算法是 HS256。

第二部分是 Payload Claim。这一部分被定义为实体的状态，就像 token 自身附加元数据一样，claim 包含我们想要传输的信息，以及用于服务器验证的信息，一般有 reserved/public/private 三类。

第三部分是 Signature。它由前面在 Header 指定的算法 HS256 加密两个参数构成，第一个参数是`经过编码的 Header 与经过编码的 Payload 通过 . 连接之后的字符串`，第二个参数是生成的密钥，会由服务器保存。每次服务器接收到 token 之后，也是先解密出用于验证的用户信息以及密钥，再与自己保存的密钥对比是否相同，以此来验证用户的身份。


写完这么一大篇真是好累…还想继续深挖一下 JWT 具体的应用、服务端如何验证、JWT 使用 cookie 存储好还是 HTML5 Web Storage 好的，剩下的只能留在下回分解啦。。

感谢以下参考文章： 
[Json Web Token Introduction](https://jwt.io/introduction/)
[使用Json Web Token设计Passport系统](https://yq.aliyun.com/articles/59043)
[深入RESTful无状态原则](http://blog.csdn.net/jmilk/article/details/50461577)
