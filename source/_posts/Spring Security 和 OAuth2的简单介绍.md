---
title: Spring Security 和 OAuth2的简单介绍
date: 2021-01-05 18:00:16
tags:
- Spring Security
- OAuth2
categories: 
- 授权
cover: /photo/src=http___mmbiz.qpic.cn_mmbiz_png_UicsouxJOkBfd0Pic2LmCJvAtFKLEOBsjibZwt6g3zKDUOJXrAjicaWiatrxPTXaQgJwP2VOZZ2d6F6TKRBHriaxqINA_640_wx_fmt=png&refer=http___mmbiz.qpic.jpeg
---


# Spring Security 和 OAuth2的简单介绍
## Spring Security
Spring Security，这是一种基于 Spring AOP 和 Servlet 过滤器的安全框架。它提供全面的安全性解决方案，同时在 Web 请求级和方法调用级处理身份确认和授权。

它是一个专注于为Java应用程序提供认证和授权的框架。像所有的Spring项目一样，Spring Security的真正威力在于它可以很容易地扩展以满足客户的需求。

做web应用时，一般都需要用到安全框架，而现在web应用中，主要有两套安全框架，就是shiro 和 spring security。

功能上两者都差不多，shiro有的功能spring security都有，而且spring security还有一些额外的功能，就是对OAuth的支持。![在这里插入图片描述](https://img-blog.csdnimg.cn/20210105114715321.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
上图是网上的一些对比，如果是做单体项目，shiro足以，如果是分布式项目，推荐spring security 和 oauth2.0。

## OAuth2
OAuth 2.0是业界标准的授权协议。OAuth 2.0注重客户端开发者的简单性，同时为Web应用、桌面应用、手机和客厅设备提供特定的授权流程

比如微信有些小程序使用的时候需要微信的账号信息，如头像昵称等，这时候就是通过OAuth授权协议将微信的头像昵称信息共享给小程序，而不需要用到微信的账号和密码。
还有现在很多网站支持第三方登陆，如支持qq,微博，github等等，这种登陆方式也是基于OAuth2的，不必担心会造成密码泄漏。

[彻底理解 OAuth2 协议](https://www.bilibili.com/video/av35979732?from=search&seid=6370600346221545740)
这个视频深入浅出，浅显易懂的讲述了OAuth2的基本概念，授权模式，应用场景，看完之后对OAuth 2会有一个简单的了解。


![](https://img-blog.csdnimg.cn/20210105145216316.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
这张图片是网上找来的，个人认为对OAuth2的流程画的很直观，网上其他关于OAuth2的详细解说有很多，感兴趣的可以去看看，这边就简单介绍一下。

