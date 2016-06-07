title: Oauth2认证
date: 2016-2-1
tags: [Java, Oauth2] #文章标签，可空，多标签请用格式，注意:后面有个空格
---
## 目的

使使用者不需要把自己的帳號密碼給予第三方服務也可以使用服務提供者的API。

## 应用场景

我想将储存在google driver上的照片使用一家网络冲洗照片公司 - 云照片冲洗出来。但是云照片没有访问我的google driver的权利。
为了使它能够访问，我必须对其进行授权。如何获取授权？最简单的方式就是告诉云冲洗我的google账户和密码，但是这种方式会有潜在的风险。
OAuth2就是为了解决授权问题而生。

## 名词解释

在OAuth2中有几个概念：
- Third-party application：第三方应用程序，又称"客户端"（client），即云照片。
- HTTP service：HTTP服务提供商，本文中简称"服务提供商"，即上一节例子中的Google。
- Resource Owner：资源所有者，本文中又称"用户"（user）。
- User Agent：用户代理，本文中就是指浏览器。
- Authorization server：认证服务器，即服务提供商专门用来处理认证的服务器。
- Resource server：资源服务器，即服务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

## 设计思路
OAuth在"客户端"与"服务提供商"之间，设置了一个授权层（authorizationlayer）。
"客户端"不能直接登录"服务提供商"，只能登录授权层，以此将用户与客户端区分开来。
"客户端"登录授权层所用的令牌（token），与用户的密码不同。用户可以在登录的时候，指定授权层令牌的权限范围和有效期。
"客户端"登录授权层以后，"服务提供商"根据令牌的权限范围和有效期，向"客户端"开放用户储存的资料。

## 流程

![process](http://7xq5i5.com1.z0.glb.clouddn.com/img_OAuth2.png "OAuth2授权流程")

- 用户打开客户端以后，客户端要求用户给予授权。
- 用户同意给予客户端授权。
- 客户端使用上一步获得的授权，向认证服务器申请令牌。
- 认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
- 客户端使用令牌，向资源服务器申请获取资源。
- 资源服务器确认令牌无误，同意向客户端开放资源。

第二步是最关键的一步，即用户怎样才能给于客户端授权。有了这个授权以后，客户端就可以获取令牌，进而凭令牌获取资源。

## 客户端的授权模式
客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0定义了四种授权方式。
- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

### 授权码模式

![process](http://7xq5i5.com1.z0.glb.clouddn.com/img_Authorization_code.png "授权码模式")

步骤如下：
- 用户访问客户端，后者将前者导向认证服务器。
- 用户选择是否给予客户端授权。
- 假设用户给予授权，认证服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。
- 客户端收到授权码，附上早先的"重定向URI"，向认证服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。
- 认证服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。

参数如下：

步骤1中，客户端申请认证的URI，包含以下参数：
- response_type：表示授权类型，必选项，此处的值固定为"code"
- client_id：表示客户端的ID，必选项
- redirect_uri：表示重定向URI，可选项
- scope：表示申请的权限范围，可选项
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

下面是一个例子。
```

GET https://github.com/login/oauth/authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

步骤3中，服务器回应客户端的URI，包含以下参数：
- code：表示授权码，必选项。该码的有效期应该很短，通常设为10分钟，客户端只能使用该码一次，否则会被授权服务器拒绝。该码与客户端ID和重定向URI，是一一对应关系。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

下面是一个例子。
```
https://callbackUrl?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
```

步骤4中，客户端向认证服务器申请令牌的HTTP请求，包含以下参数：
- grant_type：表示使用的授权模式，必选项，此处的值固定为"authorization_code"。
- code：表示上一步获得的授权码，必选项。
- redirect_uri：表示重定向URI，必选项，且必须与A步骤中的该参数值保持一致。
- client_id：表示客户端ID，必选项。

下面是一个例子。

```
POST https://github.com/login/oauth/access_token?grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

步骤5中，认证服务器发送的HTTP回复，包含以下参数：
- access_token：表示访问令牌，必选项。
- token_type：表示令牌类型，该值大小写不敏感，必选项，可以是bearer类型或mac类型。
- expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
- refresh_token：表示更新令牌，用来获取下一次的访问令牌，可选项。
- scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。

下面是一个例子。
```
Accept: application/json
{"access_token":"e72e16c7e42f292c6912e7710c838347ae178b4a", "scope":"repo,gist", "token_type":"bearer"}
```

从上面代码可以看到，相关参数使用JSON格式发送（Content-Type: application/json）。此外，HTTP头信息中明确指定不得缓存。
授权码模式（authorization code）是功能最完整、流程最严密的授权模式。它的特点就是通过客户端的后台服务器，与"服务提供商"的认证服务器进行互动。对于其他模式，不在这里一一叙述。
如果大家感兴趣可以参考github的Oauth2授权流程：https://developer.github.com/v3/oauth/。

## 更新令牌

如果用户访问的时候，客户端的"访问令牌"已经过期，则需要使用"更新令牌"申请一个新的访问令牌。
客户端发出更新令牌的HTTP请求，包含以下参数：
- granttype：表示使用的授权模式，此处的值固定为"refreshtoken"，必选项。
- refresh_token：表示早前收到的更新令牌，必选项。
- scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数，则表示与上一次一致。
