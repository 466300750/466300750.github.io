---
layout: post
title: Authentication and Authorization
date: 2018-09-13 10:42:31
categories: Security
share: y
excerpt_separator: <!--more-->
---



<!--more-->

Auth0使用的业界标准：

现今网络上制订了几种不同的身份认证行业标准来规范网站应用实现用户认证及授权。这些标准是一套开放的规范和协议，告诉你如何设计一个认证和授权系统。 他们告诉你如何管理身份，安全地移动个人数据，并决定谁可以访问应用程序和数据。以下是Auth0可供选用的标准。

1. OAuth 1
2. OAuth 2
3. Open ID Connect
4. JSON Web Tokens
5. SAML
6. WS-Federation

## OAuth
### OAuth Roles
OAuth defines four roles:

- Resource Owner
- Client
- Resource Server
- Authorization Server

![](../images/oauth_abstract_flow.png)

### 企业微信网页授权登录

[详情参考文档](https://work.weixin.qq.com/api/doc#10028)

前端请求URL：

```
const authUri = 'https://open.weixin.qq.com/connect/oauth2/authorize?appid=${corpid}&redirect_uri=${encodeURIComponent(config.authUrl)}&response_type=code&scope=snsapi_base&agentid=${agentid}&state=${corpid + '--' + encodeURIComponent(location.href)}#wechat_redirect'

authUrl: '${location.origin}/api/enterprisewechat/webpage/auth/redirect'
```

redirect 到后端:

```
@GetMapping("/publicwechat/webpage/auth/redirect")
    @Transactional
    public String redirect(@RequestParam("code") String code,
                           @RequestParam("appid") String publicWechatId,
                           @RequestParam("state") String redirectUrl,
                           HttpServletResponse servletResponse) {

        if (!distributorRepository.existsByPublicWechatId(publicWechatId)) {
            throw new CommonInternalErrorException("服务号还未授权给任何经销商.");
        }

        if (!redirectUrl.startsWith(b2BProperties.getBaseUrl())) {
            throw new CommonBadRequestException("重定向域名错误.");
        }

        String openId = publicWechatUserService.retrieveOpenIdAfterAuthRedirect(code, publicWechatId);

        Optional<PublicWechatUser> factoryUser = publicWechatUserRepository.activeUserFor(openId, publicWechatId);

        if (!factoryUser.isPresent()) {
            return returnLoginRedirect(publicWechatId, redirectUrl, openId);
        }

        PublicWechatUser theUser = factoryUser.get();

        if (isFactoryAccountChanged(theUser)) {
            return returnLoginRedirect(publicWechatId, redirectUrl, openId);
        }

        JwtToken jwtToken = publicWechatJwtService.generateJwtToken(theUser);

        addTokenToCookie(servletResponse, jwtToken, "token-" + theUser.getPublicWechatId());

        return redirectTo(redirectUrl);
    }

```

后端获取openId：

```
private static String ACCESS_TOKEN_URL = "https://api.weixin.qq.com/sns/oauth2/component/access_token?appid={appId}&code={code}&grant_type=authorization_code&component_appid={componentAppId}&component_access_token={acceccToken}";


public String retrieveOpenIdAfterAuthRedirect(String code, String appId) {
        String componentAccessToken = componentAccessTokenHolder.accessToken().orElseThrow(() -> new RuntimeException("No component access token available."));

        String componentAppId = thirdPartyProperties.getAppId();
        String url = fromHttpUrl(ACCESS_TOKEN_URL).buildAndExpand(appId, code, componentAppId, componentAccessToken).toString();
        String response = restTemplate.getForObject(url, String.class);

        Map map = objectMapper.readValue(response, Map.class);
        return (String) map.get("openid");
    }
```

其中accessToken是定期刷新并存入数据库：

```
private static String COMPONENT_ACCESS_TOKEN_URL = "https://api.weixin.qq.com/cgi-bin/component/api_component_token";
```

获取到openId后获取用户信息，如果用户不存在，或者用户信息被更新过，则返回前端跳转到登录界面的redirectURL。如果微信已登录，则生成token并通过cookie加到response中。
## 用通俗的例子解释OAuth和OpenID的区别

[该例子转自该链接](https://www.cnblogs.com/linyc/p/4338092.html)

## OpenID vs OAuth

- 目的不同
	- OpenID 侧重验证.（OpenID用于不同消费者之间共享单一身份，比如淘宝也可用微博账号登录）
	- OAuth 侧重授权.（比如我们登录淘宝用微博账号时，在微博授权界面，会问你是否共享xxx）
- 不相互排斥
- 共同点
	- 开源——社区驱动
	- 包含第三方
	- 包含在consumer和service provider之间移动users
	- 包含laying a claim that is verified by the service/identity provider
		- OpenID: "I own this URL"
		- OAuth: "I own this resource"

## OpenID vs SAML2

OpenID and SAML2 都是基于联合身份概念，一下是不同点：

- SAML2 支持单点登出，OpenID 不支持
- SAML2 service providers 与 SAML2 Identity Providers 紧耦合, 但是 OpenID 依赖的机构不与 OpenID Providers紧耦合. OpenID 有一个发现协议可以动态发现相应的 corresponding OpenID Provider. SAML 的发现协议基于 Identity Provider Discovery Service Protocol.
- With SAML2, 用于与 SAML2 IdP绑定 - 你的 SAML2 身份只对 SAML2 IdP 有效. 但是对于 OpenID, 你拥有你自己的身份而且你可以映射到任何你希望的 OpenID Provider.
- SAML2 用多种绑定，但是 OpenID 只能绑定HTTP
- SAML2 可以作为 Service Provider (SP) initiated 或者 Identity Provider (IdP) initiated. 但是 OpenID 只能是 SP initiated.
- SAML 2 is based on XML while OpenID is not.

## HMAC Authentication (Hash-based Message Authentication Code)

通过hash函数和双方共享的secret key一起计算一个message authentication code. 主要用于完整性校验, 验证, 消息发送至身份识别.

简言之，服务器提供给客户端一个public APP Id and shared secret key (API Key – shared only between server and client), 这个过程只发生在客户端第一次注册到服务器。

客户端和服务器对API Key达成一致后，客户端创建一个唯一的hash表示发送到server的请求来自于自己。通过组合 request data ，它通常包含 (Public APP Id, request URI, request content, HTTP method, time stamp, and nonce) 来生成一个唯一的 hash by using the API Key.然后客户端把hash和请求发送到服务器.

一旦server收到后，尝试通过api key和request data重建hash。通过比较两个hash值来验证。

### Flow on the client side:

- 客户端组装以下内容为一个字符串：APP Id, HTTP method, request URI, request time stamp, nonce, pay load的base64编码。
- 注意：request time stamp是用Unix time表示的，可以避免时区问题。Nonce 是一个只被使用一次的任意数字或者字符串。
- 客户端对这个字符串进行hash，用api key为参数，得到的hash值是这个请求的签名。
- 签名在Authorization头部中进行传送，类似“amx”, Authorization头部中也包含APP id,request time stamp, nonce，通过：分割，格式如下：`[Authorization: amx APPId:Signature:Nonce:Timestamp]`
- 客户端发送请求

### Flow on the server side:

- server接收数据
- 从Authorization头部中获取数据
- server 在secure repository中通过查找APP id获得该客户端的API Key 
- 通过request time stamp 和 nonce 可以避免重放攻击
- server重建字符串，包含收到的所有数据，通过追加与客户端相同的顺序和编码方式的参数
- 对重建出的字符串进行hash，使用第三步中获取的api key
- 然后比对两个签名