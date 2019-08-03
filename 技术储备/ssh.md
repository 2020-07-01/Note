<!-- TOC -->

- [1. 引言](#1-引言)
- [2. ssh的加密原理](#2-ssh的加密原理)
- [3. 如何在本地生成密钥对(公钥和私钥)？](#3-如何在本地生成密钥对公钥和私钥)
- [4. 基于秘密的安全验证过程](#4-基于秘密的安全验证过程)

<!-- /TOC -->

# 1. 引言
ssh协议在平时使用非常多，比如在mysql连接，远程服务器的连接，github的连接，但是仅仅会通过他人的教程来使用ssh来实现功能，但是对其中的原理一知半解，最近在使用ssh进行远程服务器的连接，在此记录ssh的使用方法以及ssh的原理。

操作系统：ubuntu19.04

* ssh协议应用场景
  
ssh协议是用来保护客户端与服务端之间的连接。所有的用户登陆验证，文件数据的传送都会经过加密，防止网络中的其他攻击。


* ssh协议的实现

使用最普遍为openSSH，适用于linux和unix

# 2. ssh的加密原理
ssh是一种协议，主要用于安全远程登陆，最常用的实现方式是openSSH
* 两种加密方式

一是**对称加密**：即发送方(加密方)和接受方(解密方)使用同一个密钥<br>
缺点：每个客户端都一个密钥，容易泄露

二是**非对称加密**：使用公钥进行加密，使用私钥进行解密，具体过程为客户端使用server端发送给自己的public key将数据进行加密，服务端使用私钥进行解密
缺点：客户端无法验证受到的公钥是否为目标server端发送的?



# 3. 如何在本地生成密钥对(公钥和私钥)？

![Alt](https://raw.githubusercontent.com/yq-debug/Note/master/%E5%9B%BE%E7%89%87/ssh-keygen%E5%88%9B%E5%BB%BAssh%E8%BF%87%E7%A8%8B.png)

使用命令ssh-keygen生成public key和private key<br>
private key文件为/root/.ssh/id_rsa<br>
public key文件为/root/.ssh/id_rsa.pub
known_hosts存储的是已经认证的远程主机的host key，


known_hosts文件的生成：

![Alt](https://github.com/yq-debug/Note/blob/master/%E5%9B%BE%E7%89%87/know_hosts%E7%9A%84%E7%94%9F%E6%88%90-.png?raw=true)

在第一次进行登陆的时候本known_hosts文件中还没有server端的host key，无法严重server的合法性，需要进行确认。确认之后会生成server端的合法hsot key。

known_hosts文件的作用：
当client向server发起请求的时候，server要验证client的请求的合法性，client也要验证server的合法性，client就是通过known_hosts中的server端的host key来验证合法性

在本地生成密钥对之后将public key存储在服务器上，private key存储在客户端



# 4. 基于秘密的安全验证过程
场景：ecs服务器配置密钥对然后本地进行登陆

1.在ecs上生成密钥对之后会将private key下载到client<br>
2.客户端向服务器发送请求 ssh root@ecs ip<br>
3.服务器接受请求之后将服务器的公钥发送给服务端，**初次请求时为了避免中间人的攻击会出现警告(请求确认是否为目标server，确认之后会将服务端的host key保存在客户端)**<br>
4.客户端使用公钥将密码加密，加密之后发送到服务端，服务端通过private key进行解密，匹配密码是否合法,如果合法则登陆成功<br>
5.建立连接之后使用对称加密







**ssh官网：https://www.ssh.com/ssh/**






