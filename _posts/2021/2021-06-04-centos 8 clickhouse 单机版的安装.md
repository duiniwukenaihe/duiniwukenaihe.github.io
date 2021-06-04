---
layout: post
title: 2021-06-04-centos 8 clickhouse 单机版的安装
date: 2021-06-04 3:00:00
category: clickhouse
tags: centos8 clickhouse
author: duiniwukenaihe
---
* content
{:toc}
# 背景：

初始clickhouse是在一次在字节跳动参加的elasticsearch大会上面知道的，过去无聊在kubernetes集群中搭建过clickhouse但是也没有系统玩过，基本还是无脑的elasticsearch跑，也没有太深入。最近时间还算充足，就想系统跑下这些东西。当然了从简单的开始。

注： 本机服务器ip 192.168.0.193

# 1. centos8 搭建clickhouse

## 1. 验证sse 4.2是否支持

参见：[https://cloud.tencent.com/developer/article/1831400](https://cloud.tencent.com/developer/article/1831400) 已经处理过proxmox虚拟化后对sse 4.2的支持

```
[root@slave1 ~]# grep -q sse4\_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported“"

SSE 4.2 supported
```

## 2. 安装clickhouse

网上无聊找到了下面yum的安装方式，直接拿来了使用了（反正现在是仅用于测试）

```
 sudo rpm --import https://repo.clickhouse.tech/CLICKHOUSE-KEY.GPG

 sudo yum-config-manager --add-repo https://repo.clickhouse.tech/rpm/stable/x86\_64

 sudo yum install clickhouse-server clickhouse-client
```

![image.png](/assets/images/2021/06-04//xakofnbrxb.png)

![image.png](/assets/images/2021/06-04//ikbd3in25e.png)

嗯 由上图也可以清晰的看到clickhouse-server配置文件目录在: /etc/clickhouse-server目录下......

## 3. 修改 clickhouse-server配置文件

进入/etc/clickhouse-server目录下，目录结构如下：

![image.png](/assets/images/2021/06-04//8v54oaf23s.png)

当前要做的就是允许其他ip的访问，默认的应该都是本机的......localhost.

#### 1.修改 config.xml

```
vim config.xml
```

![image.png](/assets/images/2021/06-04//8ghvg55azs.png)

![image.png](/assets/images/2021/06-04//zfv924wwob.png)

#### 2. 查看users.xml

networks用ip ::/0了 应该貌似不用修改了吧？先采用默认的了

![image.png](/assets/images/2021/06-04//9fvfs4oqo7.png)

## 4.启动clickhouse-serve

```
clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

草率了，出现如下报错：clickhouse默认是用非root用户启动的！

![image.png](/assets/images/2021/06-04//3vlyrh9qd1.png)

再来一下

```
sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

![image.png](/assets/images/2021/06-04//3z4mdavia7.png)

又报错了！这是由于第一次没有sudo -u 切换用户启动 log目录下生成的log文件权限是root造成的，到log目录下chown或者chmod一下文件的权限：

```
cd /var/log

chown clickhouse.clickhouse -R clickhouse-server/
```

重新启动clickhouse：

```
sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

正常启动了......如下：

![image.png](/assets/images/2021/06-04//xfpq5tgqfg.png)

查看一下端口的启动状态，用客户端登陆一下clickhouse：

```
clickhouse-client --host=192.168.0.193 --port=9000
```

如下登陆成功：

![image.png](/assets/images/2021/06-04//e97ri2pjom.png)

其他的后面去深入吧！

# 关于后续：

## 1. 账号密码的设置

## 2. 资源的隔离控制

## 3. 集群的搭建

## 4. clickhouse的正确使用