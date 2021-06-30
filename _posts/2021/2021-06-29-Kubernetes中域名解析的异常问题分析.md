---
layout: post
title: 2021-06-29-Kubernetes中域名解析的异常问题分析
date: 2021-06-29 2:00:00
category: kubernetes
tags: kubernetes dns
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
自建kubernetes1.16集群，服务应用早期多为php应用。docker封装nginx+php-fpm基础镜像，将代码打包成image jenkins进行ci/cd构建。
php应用中出现大佬域名解析失败的报错.....what?开始怀疑过kubernets版本问题，也怀疑过网络组件。但是未能找到原因。今天正好百度搜索资料时候偶然看到：[https://www.it1352.com/589254.html](https://www.it1352.com/589254.html)，看到他上面解决的curl调取花费时间过长的时候curl指定了CURL_IPRESOLVE_V4。就顺便想了下...是了。我的集群没有禁用ipv6!划重点了：
​

**如果开启了IPv6，curl默认会优先解析 IPv6，在对应域名没有 IPv6 的情况下，会等待 IPv6 dns解析失败 timeout 之后才按以前的正常流程去找 IPv4**
# 关于解决方案:
**自己简单想一想也有两种解决方式：**

1. work节点禁用ipv6
2 php代码指定CURL_IPRESOLVE_V4
# 入手解决：

## 1.关于work节点禁用ipv6

参照：[https://blog.csdn.net/wh211212/article/details/80996364](https://blog.csdn.net/wh211212/article/details/80996364)
我是直接sysctl设置禁用IPv6的方式了，不想重启集群节点！

```
在/etc/sysctl.conf中添加以下行
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# 或者执行
sed -i '$ a\net.ipv6.conf.all.disable_ipv6 = 1\nnet.ipv6.conf.default.disable_ipv6 = 1' /etc/sysctl.conf

要使设置生效，请执行
sysctl -p
```
## 2. 关于php代码的修改
参照：[https://www.jb51.net/article/39788.htm](https://www.jb51.net/article/39788.htm)，直接扔给php小伙伴了....毕竟我也不会php。
​

# 其他可以参考的：
## 1. [k8s – coredns禁用ipv6解析](https://yuerblog.cc/2019/09/13/k8s-coredns%E7%A6%81%E7%94%A8ipv6%E8%A7%A3%E6%9E%90/)
## 2. [容器中使用nscd缓存优化 DNS 解析](https://my.oschina.net/u/2322202/blog/3115148)