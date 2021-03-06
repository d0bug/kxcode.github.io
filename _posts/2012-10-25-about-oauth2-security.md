---
layout: post
title: OAuth2安全问题的一些思考
description: "thoughts of auth2 security"
modified: 2016-04-10
tags: [oauth, thoughts]
---

OAuth是一种简单开放的授权协议，使得第三方应用可以在用户授权的情况下获得accessToken，随后第三方应用用它访问用户的资料，而无需获得用户的账户密码。

目前国内大部分互联网公司提供的API都通过OAuth进行第三方应用授权。如：豆瓣、腾讯、新浪等。另外利用微博账号、SNS账号等的快速登录，可以使中小型网站轻松获得新用户，用户无需注册流程直接登录，提升用户登录率。

刚接触OAuth时，有点不靠谱的感觉，虽然说OAuth授权不用我们透露自己的账号，但是授权之后第三方应用不用我们的账号就可以直接读写我们的信息。而很多小应用往往会期望获得远远超过自身需要的权限，这种感觉就好像有点引狼入室的感觉。谁知道那些应用什么时候会窃取我们的个人资料干些什么勾当，发广告还是小的。

> 对于开发者开说，尽量获取到用户账户的使用权限似乎是一种‘追求’，而不管用不用得到，这不禁让人想起了 Android 移动应用上的普遍高权限。

最近在freebuf上看到了一个关于OAuth2安全性的讨论。
对于一些提供快速登录的网站，往往会将微博等账号与该网站账户进行绑定的功能，这样就有可能存在一些CSRF隐患。
这里大致说一下OAuth2的授权流程：

第三方应用向服务提供商请求授权，将用户重定向到该授权页面。提交到该页面的参数包括

- client_id：标志该应用的ID
- redirect_uri：授权后的回调地址
- response_type：code 或者 token
- scope：申请权限的范围.
- state：用来维护请求和回调状态的附加字符串，在授权完成回调时会附加此参数，应用可以根据此字符串来判断上下文关系。

用户在该授权页面上选择授权与否，如果授权的话再重定向到回调页面，并返回一个authorization_code
应用再通过该authorization_code向服务提供商请求accessToken

存在一种攻击情景是，使用OAuth机制+能够向配置中添加OAuth提供者的信息，攻击OAuth：

>1. 找到一个第三方应用（网站）A，点击“使用B网站账号登录”。然后用你的B网站账号登录一下，点击授权后，拦截数据包。你需要从B网站那里得到回调但先不要访问它。
>2. 不要访问回调的URL(类似于：http://pinterest.com/connect/facebook/?code=xxxxxxxx),仅仅把它放在<img src=”URL”>或<iframe>标签下保存起来就可以了。
>3. 现在你需要做的是诱使用户(某一特定的用户或者A网站上的随机的用户)点击该链接，发送HTTP请求到你的callback URL。
目标必须在发送HTTP请求时处于登录状态。 做得好，你的B网站账户已经和目标在A网站上的账户连接上了。
现在，按下“Log In with that OAuth Provider”-你现在直接可以登录到目标在A的账户上了。享受吧：读取私信，发送评论，更改支付细节额，做任何你想做的事情。
当你玩儿够了之后你只要和那个OAuth提供商断开并登出就可以了。

攻击思路是攻击者将自己的服务提供商的账号进行授权，到第二步进行重定向时终止，然后实施类似于CSRF的攻击，将该URL发送给一个已经登录第三方网站的受害者，使得他点击后完成OAuth流程，最后攻击者的授权账号与受害者第三方网站账号进行了绑定，最后攻击者可以对其账号进行非法操作。如果网站没有使用判断上下文环境的参数state，而且回调地址是不存在随机hash的固定地址，那么可能存在这种风险。博文中也列出了很多漏洞站点。

根据文中步骤对国内一些网站进行了少量的测试。结果显示，国内站点已经考虑到了这方面的安全风险并做了一定的措施。

115网盘提供的使用第三方合作伙伴豆瓣账号登陆的功能，在同意授权之后，重定向回115时，即使用户之前处于登陆状态，也会要求用户再次输入账号密码，才能绑定。
虾米使用QQ账号登陆，实施这种CSRF时会有openid错误提示。

说到底，OAuth也只是进行授权(authorization),而非认证(authentication)。很多风险也是由于开发者将授权与认证相混淆导致的。OAuth本身也提供了一个用于判断上下文的参数state，对于这里的state可以使用一个随机hash值，网站对账号添加绑定时，可以验证session[state]与提交的state参数，或许会使用户体验有所提升,不用进行很多的授权和登陆动作。

**Reference**

<http://www.freebuf.com/articles/web/5997.html>

<http://www.freebuf.com/articles/1381.html>

<http://homakov.blogspot.com/2012/07/saferweb-most-common-oauth2.html>
