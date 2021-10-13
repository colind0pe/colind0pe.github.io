---
title: HackTheBox-Oopsie
date: 2021-08-2 17:05:21
categories:
- 渗透测试
tags:
- HackTheBox
---

## 实验环境

![](https://i.loli.net/2021/07/27/Hd9VUzsDhXpJxSb.png)

## 信息收集

**端口扫描：**

```
nmap -sT -Pn 10.10.10.28
```

![](https://i.loli.net/2021/07/27/ZD8N9JvCn3Gtkyc.png)

发现80端口开放，在浏览器打开，啥也没有，尝试找登录界面

![](https://i.loli.net/2021/07/27/bexZV7YuQCfHnJW.png)

**目录扫描：**

成功找到登录界面

![](https://i.loli.net/2021/07/27/apWtD5Xlgq4KrkH.png)

在bp中也可以看到登录页的js加载记录

![](https://i.loli.net/2021/07/27/2RtDTbzapGBCWhv.png)

在浏览器打开

![](https://i.loli.net/2021/07/27/5NhiCuwZTcWxtLd.png)

## 实验过程

账号为`admin`，密码是上一题中出现过的`MEGACORP_4dm1n!!`，直接登录，登录后看到有一个上传功能，点击提示需要super admin权限

![](https://i.loli.net/2021/07/27/ExH9yqfplFRc6be.png)

先看一下我们现在账号信息，在url中还可以看到目前的用户id=1

![](https://i.loli.net/2021/07/27/c4GRbMgQoK1pqf9.png)

**再抓包看一下**：

id和user、role对应，可以尝试遍历用户

![](https://i.loli.net/2021/07/27/NQDFYJsuVKjvtUG.png)

**发送到interlude模块**，标记id参数，设置好payload开始爆破

![](https://i.loli.net/2021/07/27/IO7bRopKTVdyeEN.png)

![](https://i.loli.net/2021/07/27/jAMXOz6hqZakGB3.png)

看到id=30的响应包的长度比较大，看一下他的响应包的内容，确认为super admin用户

![](https://i.loli.net/2021/07/27/bH7zidyqvxF3sQL.png)

![](https://i.loli.net/2021/07/27/Z2itEGU1TQspdwe.png)

修改目前用户的user和role值，成功越权到super admin用户

![](https://i.loli.net/2021/07/27/D4Zw8Bkz39QbPyT.png)

![](https://i.loli.net/2021/07/27/EM872nWlGr6Qcux.png)

然后尝试上传shell，同样修改当前用户的user和role值，就可以进行文件上传

![](https://i.loli.net/2021/07/27/qmQXoHtZi2lV1By.png)

直接用kali自带的反弹shell文件，修改好ip和port

![](https://i.loli.net/2021/07/27/dqBTIuL8E36UJXf.png)

注意上传的过程中也要修改当前用户的user和role，改成super admin的值，就可以成功上传

![](https://i.loli.net/2021/07/27/zmrJRpM73eCjV9q.png)

![](https://i.loli.net/2021/07/27/hnu7MqHeKplP8oF.png)

接下来开启一个监听端口来接收反弹的shell

![](https://i.loli.net/2021/07/27/qW8BbmrcNsvJwoP.png)

然后就是要找到我们上传的shell，扫目录的时候有一个uploads的路径，我们上传的文件应该就在这里

**访问上传的shell来执行**

![](https://i.loli.net/2021/07/27/Kcw1hkTyXuig4r9.png)

回到netcat中就可以看到反弹的shell了

![](https://i.loli.net/2021/07/27/TZQSK9vkXonWIfz.png)

拿到shell先看看有没有有用的文件，我们拿到了数据库的账号密码

![](https://i.loli.net/2021/07/28/wFjIotzpmd9gQBC.png)

但是现在的shell是非交互式的，我们接下来要升级到交互的shell，两者的区别如下：

```
交互式模式就是shell等待你的输入，并且立即执行你提交的命令，退出后才终止
非交互式模式就是以shell script方式执行，shell不与你进行交互，而是读取存放在文件中的命令并执行它们，读取到结尾就终止
```

```
用netcat获得的shell是非交互式的，不能传递tab来进行补全，不能使用su、nano，也不能执行ctrl+c等命令，所以我们需要升级为交互式的shell
```

逐条键入命令：


```
# 将在环境变量下将shell设置为/bin/bash且参数为-q和/dev/null的情况下运行脚本，-q参数为静默运行，输出到/dev/null里，如果不加script -q /dev/null不会新启一个bash，shell=/bin/bash只是设置shell为bash，加了以后会给你挂起一个新的shell，并帮你记录所有内容
SHELL=/bin/bash script -q /dev/null
# 将netcat暂挂至后台
Ctrl-Z
# 将本地终端置于原始模式，以免干扰远程终端
stty raw -echo
# 将netcat返回到前台，注意：这里不会显示输入的命令
fg
# 重置远程终端，经测试也可以不进行此操作
reset
# 运行xterm
xterm
```

拿到交互的shell我们就可以切换到Robert用户了

![](https://i.loli.net/2021/07/28/EdiQx9hAYpDysSk.png)

**获取普通用户权限的flag**

![](https://i.loli.net/2021/07/28/aEARfWQUJsmChMP.png)

## 权限提升

下面我们就要想办法提权，我们先看看这个组里面有没有特殊权限

```
# -type f 为查找普通文档，-group bugtracker 限定查找的组为bugtracker，2>/dev/null 将错误输出到黑洞（不显示）
find / -type f -group bugtracker 2>/dev/null 
# -al 以长格式方式显示并且显示隐藏文件
ls -al /usr/bin/bugtracker           
```

![](https://i.loli.net/2021/07/28/KLmAGangISYdcpb.png)

![](https://i.loli.net/2021/07/28/AtRHJfpn6IU7xMg.png)

拥有者有`s`（`setuid`）特殊权限，可执行的文件搭配这个权限，可以得到特权，任意存取该文件的所有者能使用的全部系统资源，我们尝试运行它，发现这个文件根据提供的`ID`值输出以该数字为编号的`bug`报告

![](https://i.loli.net/2021/07/28/y82ifUD4bzuLOTo.png)

接下来我们可以使用`strings`命令来看看对象文件或二进制文件中查找可打印的字符串

![](https://i.loli.net/2021/07/28/KLOeI9juaobwQ1W.png)

可以看到`bugtracker`调用了`cat`命令，输出了`/root/reports/`目录下的`bug`报告，其实本来我们当前用户是没有权限访问`/root`目录的，但是我们有了`setuid`后就拥有了`/root`目录的访问有权限，也就拥有了`root`权限，当前用户执行`bugtracker`程序是会优先使用当前的`path`变量，这时候我们就可以在当前用户环境变量`指定的路径`中搜索`cat`命令，然后创建一个恶意的`cat`命令，修改当前用户环境变量，完成提权操作

```
export PATH=/tmp:$PATH                //将/tmp目录设置为环境变量
cd /tmp/                            //切换到/tmp目录下
echo '/bin/sh' > cat                //在此构造恶意的cat命令
chmod +x cat                        //赋予执行权限
```

![](https://i.loli.net/2021/07/28/ghfboUQt6Oe1wd7.png)

这样`bugtracker`再次调用`cat`命令时实际上调用的是`/tmp`目录下的恶意的`cat`命令，我们运行一下`bugtracker`可以看出，此时`robert`用户临时具有了`root`权限，执行`id`命令发现只是`robert`用户的`uid`变为了`root`，不是真正的`root`用户

![](https://i.loli.net/2021/07/28/SbzZWCsydVNm5t9.png)

这样我们就可以获取system的flag了

![](https://i.loli.net/2021/07/28/hs5lVjfuewa1ALJ.png)

在/root/.config/filezilla/filezilla.xml文件中有下一题的ftp账号密码

![](https://i.loli.net/2021/07/28/CEoznHSvYQuDI7F.png)

> 参考：
>
> 1、https://www.echocipher.life/index.php/archives/872/
>
> 2、[关于Linux下s、t、i、a权限](https://www.cnblogs.com/qlqwjy/p/8665871.html)
>
> 3、[LINUX s权限位提权](http://www.361way.com/suid-privilege/5965.html)

