---
title: Oauth2.0 实现短信验证码登录过程详解
date: 2021-01-07 18:30:16
tags:
- Spring Security
- OAuth2
categories: 
- 登录
cover: /photo/src=http___img2018.cnblogs.com_blog_1158242_201812_1158242-20181220161647691-594119337.png&refer=http___img2018.cnblogs.jpeg
---


# Oauth2.0 实现短信验证码登录过程详解

## 前言
公司新分配了微服务的项目，要求在已有密码登陆的架构上增加手机验证码登陆功能，原有的密码登陆都没摸清的我懵逼了，百度看了很多Spring Security 和 OAuth2基本概念还有一些如何实现短信验证码登录功能的博客，很多文章一上来就给你展示n个类的，或者直接贴上一大堆代码的，对于才了解一点基本概念的我来说属实有点难啃，好在找到了一位大佬将思路以及实现写的很清晰的文章，鉴于大佬的文章图片挺模糊的，我把这几天弄出来的功能简单整理一下，做个记录。(不想看分析，想直接简单粗暴开始干的请直接跳到标题为编码阶段的开始看)

阅读本文需要的基础知识：

 - 熟练掌握Java
 - 掌握了Spring Boot基础知识
 - 默认已经整合好密码模式

### 架构搭建

本文只说验证码登录相关部分，默认大家Spring Cloud OAuth2这部分环境已经搭建好。
[Spring Cloud OAuth2 实现用户认证及单点登录](https://www.cnblogs.com/fengzheng/p/11724625.html)
## 需求
OAuth 2 有四种授权模式，分别是授权码模式（authorization code）、简化模式（implicit）、密码模式（resource owner password credentials）、客户端模式（client credentials）。

[想了解简单的Spring Security 与 OAuth2的基本概念和应用场景的可以点这里](https://princeling66.github.io/MyBlog/2021/01/05/Spring%20Security%20%E5%92%8C%20OAuth2%E7%9A%84%E7%AE%80%E5%8D%95%E4%BB%8B%E7%BB%8D/)

但是有时候我们需要一些自己特殊的模式登录，比如说验证码登录，第三方登录等等。上面几种模式并不是很方便，下面是将密码模式改造成手机验证码的方式来登录。

## 思路分析

## TokenEndpoint类中可以看到入口/oauth/token
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106171948580.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
首先我们要知道/oauth/token是我们登录验证调用的接口，无论是get还是post都会走到post这个方法中。

其次我们需要打断点去看一下源码，了解两件事。

1、哪一步加入的四种授权模式。（因为四种授权模式是写死在源码里的，拓展的时候我们得自己加上）

2、密码模式是在哪里开始验证用户名密码。（为的是改造那部分来写自己的验证码模式）

## 看源码阶段
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106172230501.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

```csharp
OAuth2AccessToken token = getTokenGranter().grant(tokenRequest.getGrantType(), tokenRequest);
```

这句代码就是我们要找的突破点

getTokenGranter()会调用tokenGranter()，tokenGranter的内容为：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106172734520.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

### getDefaultTokenGranters()

就是这个方法，写死了那四种授权模式。红圈部分是在添加密码模式，之后我们就是要在这里添加我们的授权码模式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106172843101.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

### 密码模式实现类ResourceOwnerPasswordTokenGranter
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106173031320.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
ResourceOwnerPasswordTokenGranter继承了AbstractTokenGranter类

而AbstractTokenGranter的实现类正好对应着四种授权模式，外加个刷新token的
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021010617325476.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
那么改造思路就很明确了

1、我们自己新增一个验证码模式类继承AbstractTokenGranter，

2、在源码调用getDefaultTokenGranters()方法的时候我们手动把这个类加进去不就行了

## 编码阶段

#### 新建自定义验证码类SmsTokenGranter

内容直接复制ResourceOwnerPasswordTokenGranter中的内容
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106173500359.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
password换成我们自定义的sms
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106173652851.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106173901371.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
重点在于checkPhoneSms(parameters);这是我们自定义的验证方法，对于验证码的验证就在这个方法中去做

```csharp
    //在这个类中无法使用@autowire来从spring容器中取bean，因为不是他管理的，
	//所以我们写了一个工具类(就是附录中的ApplicationContextAwareUtil)去取我们需要的bean
    public UserClientTransitService userClient = ApplicationContextAwareUtil.getBean("userClientTransitServiceImpl");

    public void checkPhoneSms(Map<String, String> parameters) {
        String phone = (String) parameters.get("username");
        String code = (String) parameters.get("code");//验证码

        if (StringUtils.isBlank(phone)) {
            throw new InvalidGrantException("手机号不能为空");
        }
        if (StringUtils.isBlank(code)) {
            throw new InvalidGrantException("验证码不能为空");
        }

        /**
         1、从缓存中根据手机号取出验证码校验不能为空
         2、验证通过后根据手机号查出用户名密码，设置到parameters中，这样本质就还是走密码模式
         （如果未创建的用户在这一步可以调用创建用户的方法，同时随机生成一个密码存着）。

         特别注意：设置password时，根据自己的实际情况决定是否要使用PasswordEncoder，否则密码验证
         时会一直报错。不清楚PasswordEncoder是啥的同学，建议先阅读下面链接的环境搭建
            https://www.cnblogs.com/fengzheng/p/11724625.html
         */
        R<UserInfo> result = null;
        result = userClient.userClientTransit(phone);
        if (result.isSuccess()) {
            UserInfo userInfo = result.getData();
            User user = userInfo.getUser();
            //设置username和password，之所以设置手机号
            //是为了在loadUserByUsername方法中和原本走密码模式的username做一个区分
            parameters.put("username", user.getPhone());
            parameters.put("password", user.getPlaintextPassword());
        }

    }
```
我这里因为是微服务，user模块需要通过feign类远程调用，这边就新建了个userClientTransitServiceImpl去调用userClient,把它交给bean管理这边就能拿到了，
之前做接口的时候数据库存了明文密码，加上我这里使用了PasswordEncoder，所以user.getPlaintextPassword()拿的是明文密码，这块之后还是需要再优化一下。
![PasswordEncoder](https://img-blog.csdnimg.cn/20210106175308601.jpg#pic_center)
PasswordEncoder.matches会把明文密码加密后和数据库中的密码对比。

loadUserByUsername方法中会根据username获取用户的权限信息组装起来，我这边因为有一个门户和一个后台的登陆，所以加了个手机登陆的类型，验证码的校验也在这里做了，也可以在上面checkPhoneSms()做
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106175624638.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
注意点：
原本的密码模式，是直接拿到我们输入的密码然后用passwordEncoder加密后再继续处理的，改成手机登录后我们拿到的就是一个加密的密码，所以它又会加密一次。所以在loadUserByUsername方法中，我们发现是其他模式登录时，把password也再加密一次。这样对比的密码就一致了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106180216691.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

### 在源码中加入我们自定义的验证码类
方式有好几种，我就提供一种简单粗暴的，有想法的同学可以自己更改方式

我这里是从源码中把AuthorizationServerEndpointsConfigurer类复制出来，放在同名的包下，这样启动项目后就会优先走我们修改的代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106180407889.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

修改的东西很简单，把密码模式那句话复制一下，把new的类从ResourceOwnerPasswordTokenGrantercoin换成我们自定义的，SmsTokenGranter类即可,参数不用改变。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106180525414.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)
### 修改OAuth2 的认证中心配置文件
加上sms，这样在后面查找授权模式的时候才能找到,我的是在OAuth2配置文件是在数据库的，有不同的可AuthorizationServerConfiguration中增加配置
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106180657738.jpg#pic_center)

## 调用测试

#### 输入一个错误的验证码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106182952734.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

#### 输入一个正确的验证码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210106183105948.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDY5OTM0OA==,size_16,color_FFFFFF,t_70#pic_center)

最后就能验证成功了

## 附录

#### ApplicationContextAwareUtil

```csharp
/**
 * 存在一些情况无法直接Autowired注入我们需要的类，通过此工具类则可以直接获取spring中的bean
 * <p>
 * 根据类名获取实例，例：
 * public StringRedisTemplate stringRedisTemplate = ApplicationContextAwareUtil.getBean("stringRedisTemplate");
 */
@Component
public class ApplicationContextAwareUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        ApplicationContextAwareUtil.applicationContext = applicationContext;

    }

    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) throws BeansException {
        if (applicationContext == null) {
            return null;
        }
        return (T) applicationContext.getBean(name);
    }

}
```

## 参考文章
[Spring Cloud OAuth2实现手机验证码登录](https://www.cnblogs.com/grimm/p/13518111.html)
[Spring Security OAuth2 开发指南（非最新版本）](https://www.cnblogs.com/xingxueliao/p/5911292.html)
[Oauth2---AuthorizationServer配置](https://blog.csdn.net/silmeweed/article/details/101603227)
[Spring Cloud OAuth2 实现用户认证及单点登录](https://www.cnblogs.com/fengzheng/p/11724625.html)
