---
title: zookeeper客户端命令详解
date: 2018-03-19 20:10:45
tags:
- zookeeper
- zookeeper客户端命令
categories:
- zookeeper
---

本篇分析zookeeper客户端常用指令

<!-- more -->

## 启动客户端

```bash
sh zkCli.sh [-server ip:port] #启动客户端
```

默认连接本机的zookeeper，如果指定server，尝试连接指定server的zookeeper

## connect

```bash
connect host:port #连接zk服务端，与close命令配合使用可以连接或者断开zk服务端
```

![](connect.png)

## help

```bash
help #查看有哪些命令
```

![](help查看命令.png)

以上是client支持的命令，注意每个命令符之间一定要有空格，两个常见的命令符做个统一说明：

**path** 表示节点的绝对路径（因为zookeeper不支持相对路径）

**watch** 表示是否对节点进行监听（节点的变更将发送客户端通知）

## ls

```bash
ls path [watch] ##查看path节点下的一级子节点
```

![](ls命令.png)

## ls2

```bash
ls2 path [watch] ##查看path节点下的一级子节点，并返回当前节点的stat信息
```

![](ls2命令.png)

## create

```bash
create [-s] [-e] path data acl ##创建子节点
```

-s 表示是否为有序节点

-e 表示是否为临时节点

path 节点的绝对路径

data 节点的数据

acl 访问权限

如果父节点不存在，无法创建，因此节点必须逐级创建。

![](父节点不存在.png)

默认情况下（不带-s 和-e参数）创建的是持久节点

![](创建节点.png)

![](创建有序节点.png)

![](创建临时节点.png)

![](重连后节点被删除.png)

## get

```bash
get path [watch] ##获取对应节点的数据和stat信息
```

![](getPath.png)

## set

```bash
set path data [version] ##设置存储的数据
```

[version] 为节点数据的版本，每次数据变更，dataVersion都会递增，version参数类似于乐观锁，指定修改对应version的节点数据，如果version不匹配，不做任何操作，不带version表示忽略版本号。

![](dataVersion递增.png)

![](set命令.png)

## delete

```bash
delete path [version] #删除节点
```

和set命令一样，delete也可以删除指定version的节点，不带version则表示忽略版本号直接删除

![](删除节点.png)

## rmr

```bash
rmr path #递归删除
```

如果当前节点下存在子节点，直接删除当前节点会提示节点非空（存在子节点）

![](存在子节点，不允许删除.png)

此时就要先删除子节点，才能删除当前节点，rmr命令允许递归删除当前节点。

![](递归删除.png)

## stat

```bash
stat path [watch] #获取节点stat信息
```

跟get的差异在于只获取stat的信息

![](stat命令.png)

## setquota 

```bash
setquota -n|-b val path ##设置配额属性
```

-n 表示该节点下子节点的个数count限制（包括当前节点自己）；

-b 表示该节点存储的数据字节大小bytes限制；

![](setquota命令.png)

## listquota

```bash
listquota path #查看配额属性
```

![](listquota.png)

之前用setquota命令设置了最大节点count=3，bytes=-1表示不限制大小（count=-1也表示不限制），stat行count=1表示当前节点数为1，bytes=3表示当前字节大小为3

## delquota

```bash
delquota [-n|-b] path  #删除配合属性
```

-n 只删除count配额

-b 只删除bytes配额

不带-n | -b表示删除该节点的所有配额属性

![](delquota命令.png)

## printwatches on|off

```bash
printwatches on|off #开启/关闭 监听信息打印
```

查看是否开启打印监听

![](printwatches命令.png)

对一个节点设置监听，然后修改节点数据

![](printwatches监听打印.png)

## setAcl 

```bash
setAcl path acl #设置节点的访问权限
```

节点的访问权限必须单独设置，子节点无法继承父节点的访问权限。上一篇有提到zookeeper的ACL权限，授权命令为**scheme : id : permission**，permission对应Create（c）、Read（r）、Write（w）、Delete（d）、Admin（a）

![](setAcl命令.png)

## getAcl

```bash
getAcl path #获取节点的访问权限
```

![](getAcl.png)

## sync

```bash
sync path #在集群中强制同步节点信息
```

## history

```bash
history #列出执行的命令历史
```

![](history.png)

## redo

```bash
redo cmdno #再次执行 id = cmdno的命令，需配合 history命令查出历史命令的id
```

![](redo.png)

## addauth

```bash
addauth scheme auth #节点认证
```

## close

```bash
close #断开与zk服务端的连接
```

![](close.png)

断开后，Client状态由CONNECTED变更为CLOSED

## quit

```bash
quit #退出客户端
```

![](quit.png)

















