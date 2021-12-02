---
title: SSH转发和iptables端口转发
date: 2021-12-02 14:54:23
categories:
- 内网渗透
tags:
- 端口转发
---

## SSH端口转发简介

SSH会自动加密和解密所有SSH客户端与服务端之间的网络数据。SSH还能够将其他TCP端口的网络数据通过SSH链接来转发，并且自动提供了相应的加密及解密服务。这一过程也被叫做"**隧道**"（tunneling），这是因为SSH为其他TCP链接提供了一个安全的通道来进行传输而得名。例如，Telnet ，SMTP ，LDAP这些TCP应用均能够从中得益，避免了用户名，密码以及隐私信息的明文传输。而与此同时，如果工作环境许中的防火墙限制了一些网络端口的使用，但是允许SSH的连接，也能够将通过将TCP用端口转发来使用SSH进行通讯。

**SSH端口转发的两大功能**：

- 加密SSH Client端至SSH Server端之间的通讯数据。
- 突破防火墙的简直完成一些之前无法建立的TCP连接。

## SSH本地SOCKS5代理

```
ssh -qTfnN -D 7777 username@remotehost
```

```
参数说明：
-C  压缩数据
-q  安静模式
-T  禁止远程分配终端
-n  关闭标准输入
-N  不执行远程命令
-f  ssh后台运行
-D  本地的端口
```

![image-20211201102041861](C:\Users\Colin\AppData\Roaming\Typora\typora-user-images\image-20211201102041861.png)

浏览器开启代理：

![image-20211201102314300](C:\Users\Colin\AppData\Roaming\Typora\typora-user-images\image-20211201102314300.png)

IP地址已经发生变化：

![image-20211201102246562](C:\Users\Colin\AppData\Roaming\Typora\typora-user-images\image-20211201102246562.png)

## SSH本地转发

**场景：**

1. 本机能与中间服务器互通
2. 中间服务器能与目标机器互通，中间服务器已拿到目标机器权限
3. 本地不能直接访问目标机器
4. 目标机器不出网
5. 目的：本机能访问目标机器

**命令：**

```
ssh -L localport:targethost:targetport username@sshserver
```

```
localport       本机开启的端口
targethost      目标机器的IP地址
targetport  	目标机器的端口
username		中间服务器的用户名
sshserver       中间服务器的IP地址
```

**结果：**

此时，在在本机访问localport就可以访问目标主机的targetport了

## SSH远程转发

反向连接的一种，可以穿透内网防火墙，在内网中比较好用

```
ssh -R sshserverport:targethost:targetport username@sshserver
```

```
sshserverpor        中间服务器的端口号
targethost          目标机器的IP地址
targetport      	目标机器的端口
username			中间服务器的用户名
sshserver           中间服务器的IP地址
```

**开启远程需要更改配置**

```
sudo vim /etc/ssh/sshd_config	
#任何人访问这台机器的某一个端口，都可以访问到目标机的映射出的端口；这个需要在中间服务器上开启
GatewayPorts    yes
```

```
sudo /etc/ssh/sshd_config restart   #重启SSH
```

**因为是反向连接，所以肯定需要在目标机器上执行命令**

在目标机器上执行命令：

```
ssh -R 8899:10.10.10.132:80 test@10.10.10.135 
#把目标机的80端口转发到10.10.10.135(中间服务器)上的8899端口
```

现在任何机器，只要访问10.10.10.13这台中间服务器的8899端口，就相当于访问了不出网的内网10.10.10.132机器的80端口

## iptables端口转发

### 基础配置

1. 修改内核文件实现端口转发

   **方法1：**

```
1.编辑sysctl配置文件 vim /etc/sysctl.conf
2.开启ipv4 forward
```

​		**方法2**： 直接sysctl修改

```
使用sysctl -w net.ipv4.ip_forward=1
然后查看sysctl -p和之前修改的一样。
```

### 本地端口转发

REDIRECT模式是防火墙所在的机子内部转发包或流到另一个端口，也就是所有接收的包只转发给本地端口。

将本机的 7777 端口转发到 6666 端口：

```
iptables -t nat -A PREROUTING -p tcp --dport 7777 -j REDIRECT --to-port 6666
```

### 远程端口转发

通过 1.168 的 6666 端口访问 1.8 的 7777 端口，在 1.168 上设置：

```
sysctl -w net.ipv4.ip_forward=1
iptables -t nat -A PREROUTING -p tcp --dport 6666 -j DNAT --to-destination 192.168.1.8:7777
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.8 --dport 7777 -j SNAT --to-source 192.168.1.168
```

### 删除该端口转发

查看当前iptables 的 nat 表的所有规则：（不用 -t 指定表名默认的是指 filter 表）

```
iptables -t nat -nL --line
```

删除指定表的指定链上的规则， -D 并指定序号即可。

```
iptables -t nat -D PREROUTING 1
```



> 参考：
>
> [SSH代理转发](https://mp.weixin.qq.com/s/tri1ruKqc-YztdWC-JmznA)
>
> [SSH端口转发详解及实例](https://www.cnblogs.com/keerya/p/7612715.html)
>
> [一文带你了解iptables用法及端口转发](https://www.freebuf.com/articles/web/289254.html)
>
> [iptables 端口转发](https://blog.csdn.net/zhouguoqionghai/article/details/81947603)
