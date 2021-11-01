---
title: driftingblues4 VulnHub Walkthrough
date: 2021-10-14 15:49:32
categories:
- 渗透测试
tags:
- VulnHub
---

#### 主机发现

```
arp-scan -l
```

<img src="https://i.loli.net/2021/10/14/zhRLGB6DEPml2uv.png" style="zoom:67%;" />

#### 端口扫描

```
nmap -sC -sV -p- 192.168.62.174
```

<img src="https://i.loli.net/2021/10/14/zLTuWXdBhYVneoM.png" style="zoom: 67%;" />

开放了21、22、80端口，先看了下21端口，发现需要密码

<img src="https://i.loli.net/2021/10/14/eoD7OHbsxQdl945.png" style="zoom:67%;" />

再访问一下80端口，看看有没有什么信息

![](https://i.loli.net/2021/10/14/RutLqmjfi95sKAS.png)

#### 信息收集

主页没有什么信息，看一下源码

![](https://i.loli.net/2021/10/14/y43jOYEQP1Wtmrz.png)

base64编码，到在线网站转换一下

<img src="https://i.loli.net/2021/10/14/iPnKaR3Hxc45Q9r.png" style="zoom:50%;" />

再转换一次

<img src="https://i.loli.net/2021/10/14/ITbCv2kKSmcjZhH.png" style="zoom:50%;" />

emmm，继续

<img src="https://i.loli.net/2021/10/14/bLqOkQABpI4UN6v.png" style="zoom:50%;" />

<img src="https://i.loli.net/2021/10/14/KkXxDfubgw8l7BE.png" style="zoom:50%;" />

最后得到一个txt的路径，访问一下

<img src="https://i.loli.net/2021/10/14/sLfinMZNFSBem8y.png" style="zoom: 67%;" />

Brainfuck加密，到在线网站解密一下

![](https://i.loli.net/2021/10/14/k8BNGWeDhpK4HcX.png)

得到了一张图片的路径，访问一下

是一个二维码

<img src="https://i.loli.net/2021/10/14/S1lyGvhiRO5VoWk.png" style="zoom: 33%;" />

扫描的结果是一张图片地址

<img src="https://i.loli.net/2021/10/14/ri9wbhBXtmn21Wv.png" style="zoom:80%;" />

访问一下

<img src="https://i.imgur.com/a4JjS76.png" alt="img" style="zoom: 67%;" />

#### 弱密码爆破

提示了几个名字，试了ssh是不支持密码登录的，那就拿来爆破ftp试试

```
hydra -L /home/colin/桌面/user -P /usr/share/wordlists/rockyou.txt ftp://192.168.62.174
```

![](https://i.loli.net/2021/10/14/TGkAwWjDZs5zoOQ.png)

成功得到了ftp的用户名和密码

用得到的用户名和密码成功登录ftp

<img src="https://i.loli.net/2021/10/14/YBD1hf4sIgpbxve.png" style="zoom:80%;" />

看了一下好像没有什么信息，但是发现这个hubert目录是有写权限的，我们可以写一个ssh的key上去，然后在本地免密登录

#### SSH免密登录

先在靶机的hubert目录下创建一个.ssh目录

![](https://i.loli.net/2021/10/14/pCZK8LhMB3wRJH5.png)

将本地的authorized_keys上传到靶机

<img src="https://i.loli.net/2021/10/14/T4Uo7cedGDLgWOC.png" style="zoom: 80%;" />

成功登录到ssh

<img src="https://i.loli.net/2021/10/14/kiIhwnUrsYcEjJu.png" style="zoom:80%;" />

现在可以获取第一个flag了

<img src="https://i.loli.net/2021/10/14/vQgpersjXcqxK7W.png" style="zoom: 67%;" />

#### 权限提升

/home/hubert目录下还有一个python脚本

![](https://i.loli.net/2021/10/14/zw8cIdiTj2N45Q6.png)

查看一下内容

<img src="https://i.loli.net/2021/10/14/f9envx2HlPkoIVh.png" style="zoom:67%;" />

执行脚本会以root权限执行一个命令，将1输出到一个文件里，去看一下这个文件

<img src="https://i.loli.net/2021/10/14/9KScPokfV6iRd4B.png" style="zoom:80%;" />

这个文件里已经有很多个1了，上面的python脚本应该是一直在执行的

将进程监控工具上传到靶机，看一下这个脚本的运行情况

先在本机起一个http服务

![](https://i.loli.net/2021/10/14/lI4oASmUELGV5y1.png)

在靶机下载，并赋予执行权限

<img src="https://i.loli.net/2021/10/14/zFAc2jdn5OeKxyR.png" style="zoom:80%;" />

发现emergency.py是每分钟执行一次的

<img src="https://i.loli.net/2021/10/14/9a6mWdXpPG34buf.png" style="zoom: 67%;" />

接下来尝试通过新建一个emergency.py来提权

<img src="https://i.loli.net/2021/10/14/Eglnj4rwqHaeAOo.png" style="zoom: 67%;" />

在本地开启一个监听端口来接收反弹的shell

![](https://i.loli.net/2021/10/14/2vdQToUR1Bs5C3b.png)

等了一会之后，本地已经接收到了shell

![](https://i.loli.net/2021/10/14/DIiKmceuLlbyo1X.png)

成功获取第二个flag

<img src="https://i.loli.net/2021/10/14/QaX6UVWEoTiO5F3.png" style="zoom:67%;" />
