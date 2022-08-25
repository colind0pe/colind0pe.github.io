---
title: 利用Tyk Gateway API和CloudFlare防火墙隐藏C2域名
date: 2022-08-25 16:35:02
categories:
- 后渗透
tags:
- Cobalt Strike
- Hiding C2 Traffic
---

## 0x01 前言

Cobalt Strike的特征已经被各大安全厂商标记烂了，加上搜索引擎爬虫，在配置C2域名后，如果不做好基础设施的隐匿，，很快会出现下图的情况。（图片截取自某开源情报社区）

![](https://s2.loli.net/2022/08/25/oZv4UrYhA8yqLcK.png)

隐藏C2的手法有很多，比如早些时候的域前置和最近热门的利用云函数隐藏。但是如今许多CDN厂商都已经禁用了域前置技术，而云函数是有免费额度的，超过之后会开始计费，许多蓝队反制帖子已经开始分享消耗云函数额度的方法了。

最近看到一篇利用Tyk Gateway API隐藏C2流量的文章，通过配置Tyk Gateway API转发恶意流量，以达到类似于域前置，或者腾讯云函数隐藏C2的效果。

在这个基础上，通过将域名托管到CloudFlare，并通过配置CloudFlare防火墙只允许Tyk Gateway API的流量访问C2域名，可以达到隐藏C2域名，并且防止搜索引擎爬虫识别Cobalt Strike特征导致域名被标记。

## 0x02 配置Tyk Gateway API

### 注册账户

访问[注册地址](https://account.cloud-ara.tyk.io/signup)，填写用户名、邮箱、密码等信息后点击注册，注册成功后选择免费版。

![image-20220825174137229](https://s2.loli.net/2022/08/25/Q58cZ3YsdFaqMhu.png)

然后设置组织名称，设置好后会提示`Deployment successful`。

### 创建并配置API 

点击`Manage APIs`后会看到如下页面：

![img](https://s2.loli.net/2022/08/25/eUybjDunGM8oJzr.png)

接下来逐一创建并配置http-get API、http-postAPI、Stager-x86 API、Stager-x64 API。以下以http-get API为例。

假设你的域名为`cslabtest.live`，且C2配置文件如下：

```
http-get {
 
    set uri "/api/v2/login";
   ....
}
 
http-post {
 
    set uri "/api/v2/status";
   ....
}
http-stager {
    set uri_x86 "/api/v2/GetProfilePicture";
    set uri_x64 "/api/v2/GetAttachment";
}
```

点击`Design new API`并填写API信息。

![img](https://s2.loli.net/2022/08/25/4m2H1xNwMij3Gu8.png)

创建好后进一步配置API，来让它能够将请求转发到我们的C2服务器。

现在我们需要更改`Listen path`和`Target URL`，TYK会监听`Listen path`的地址，并将请求转发到`Target URL`。

![img](https://s2.loli.net/2022/08/25/9u8Bsjh6cSMCfFQ.png)

为了能够上线CS，还要配置`Rate Limiting and Quotas`。都选择disable即可。

![img](https://s2.loli.net/2022/08/25/sbpULn2YTPhvwyx.png)

然后来到`Advanced Options`，取消勾选`Enable caching`。

![img](https://s2.loli.net/2022/08/25/ax9wSqE2el6nRJ7.png)

按这个步骤逐一新建http-get API、http-postAPI、Stager-x86 API、Stager-x64 API。

### 设置访问验证策略

将上一步新建的API的`Authentication`更改为`Basic Authentication`，如下所示：

![img](https://s2.loli.net/2022/08/25/1KWilQdpevuoM7y.png)

然后来到`Policies`新建策略，选择你新建的四个API。

![image-20220825181321725](https://s2.loli.net/2022/08/25/l1HnpmuLrQF6tIX.png)

然后点击`Global Limits and Quota`，确认禁用`Rate Limiting`。

![img](https://s2.loli.net/2022/08/25/NRX1zoLMPQjOF9f.png)

接着配置策略名称并设置密钥过期时间。

![img](https://s2.loli.net/2022/08/25/3s257tH9WXQuK8B.png)

点击`Create Policy`以保存新策略，之后可以在`Policies`看到它：

![img](https://s2.loli.net/2022/08/25/6GE1ygYCP753wdL.png)

### 配置访问验证Key

来到`Keys`，点击`ADD KEY`，然后在`Apply policy`选择我们之前创建的策略，并选择API。

![image-20220825182202475](https://s2.loli.net/2022/08/25/kMVjDTvq6t59m7x.png)

最后来到`Authentication`输入需要设置的用户名、密码，这里使用`test:testtesttest`作为用户名、密码。

![img](https://s2.loli.net/2022/08/25/IVfpUkga6KrndHM.png)

`Key`创建成功后会有如下提示：

![img](https://s2.loli.net/2022/08/25/OuLAM4y9PdHwSG2.png)

### 配置C2配置文件

由于上一步我们设置了访问验证，所以要在C2配置文件中添加一个请求头才能正常上线CS。

`Authorization`的请求头设置格式如下：

```
Authorization: Basic base64(username:password)
```

所以按上一步添加的`Key`，我们要在C2配置文件中添加如下请求头：

```
Authorization: Basic dGVzdDp0ZXN0dGVzdHRlc3Q=
```

最终配置文件如下：

```
http-get {
 
    set uri "/api/v2/login";
    
    client {
        header "Authorization" "Basic dGVzdDp0ZXN0dGVzdHRlc3Q=";
}
   ....
}
 
http-post {
 
    set uri "/api/v2/status";
    
    client {
        header "Authorization" "Basic dGVzdDp0ZXN0dGVzdHRlc3Q=";
}
   ....
}
http-stager {
    set uri_x86 "/api/v2/GetProfilePicture";
    set uri_x64 "/api/v2/GetAttachment";
    
    client {
        header "Authorization" "Basic dGVzdDp0ZXN0dGVzdHRlc3Q=";
}
	....
}
```

### 效果

此时直接访问我们设置的API的地址都是需要验证的，而CS是可以正常上线的。

![img](https://s2.loli.net/2022/08/25/SvRAo7Zy4WBIYxT.png)

## 0x03 配置CloudFlare防火墙

### 获得tyk.io特征

在CF的`WAF`中创建一条防火墙规则，如下：

![image-20220825184109796](https://s2.loli.net/2022/08/25/e5ZWfulg9JhKOY2.png)

然后生成一个木马尝试上线CS，这是肯定是无法上线的，来到CF的概述中可以看到所有拦截记录，点击单挑拦截记录查看详细。

![image-20220825184546311](https://s2.loli.net/2022/08/25/5X98QebcMgPF2Yv.png)

这里有很多的特征可以加到`WAF`拦截规则中，来实现只有`Tyk Gateway API`转发过来的流量才能允许访问，其他的流量都会阻止。

### 编辑防火墙规则

这里我以`ASN`为例子，配置如下：

![image-20220825184917062](https://s2.loli.net/2022/08/25/wodcvM8ufmgTYWQ.png)

保存防火墙规则即可。

## 0x04 最终效果

直接访问设置的Tyk Gateway API的地址是需要验证的。

![image-20220825185304275](https://s2.loli.net/2022/08/25/1W3mjGpAMBswySF.png)

直接访问我们的C2域名会被CF拦截。

![image-20220825185438993](https://s2.loli.net/2022/08/25/KUnbLxXzH5ECNvp.png)

CS生成木马，可以正常上线和执行命令。

![image-20220825185815180](https://s2.loli.net/2022/08/25/1igUo76aJxB4IM5.png)

## 0x05 参考

[Oh my API, abusing TYK cloud API management to hide your malicious C2 traffic](https://shells.systems/oh-my-api-abusing-tyk-cloud-api-management-service-to-hide-your-malicious-c2-traffic/)
