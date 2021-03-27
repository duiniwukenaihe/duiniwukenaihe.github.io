---
layout: post
title: Foudation-Node Metadata  Protection--节点的元数据保护
date: 2021-03-13 10:00:00
category: cks
tags:  kubernetes cks Metadata  Protection
author: duiniwukenaihe
---
* content
{:toc
![image.png](https://img-blog.csdnimg.cn/img_convert/7fc6610f45bb751ee6fee8a34c93a3cf.png#align=left&display=inline&height=291&margin=[objectObject]&name=image.png&originHeight=582&originWidth=1129&size=451386&status=done&style=none&width=564.5)
## 1. 关于元数据
kubernets集群不管是运行与公有云还是私有云，都是有些元数据的资源的各种各样的标签。比如镜像id，网络设备id,硬盘的唯一id等。
## 2. 举一个例子
### 2.1 cloud platform node metadata   云平台节点元数据

1. 拿谷歌云和亚马逊云来说
1. 默认的情况下可以从虚拟机vm（云主机）访问元数据服务的api
1. 元数据中保护有vm节点（云主机）的各种凭据信息。如网络id,镜像id  vpcid.硬盘等待各种相关信息。具体详细度要看云商平台或者私有云架构
1. 可以包含诸如kubelet凭证之类的置备数据

![image.png](https://img-blog.csdnimg.cn/img_convert/28de83482a4d86eb6e3d62a6479c0ce3.png#align=left&display=inline&height=290&margin=[objectObject]&name=image.png&originHeight=579&originWidth=1123&size=203486&status=done&style=none&width=561.5)
### 2.2 access sensitive node metadta    访问敏感节点元数据的原则


2.2.1 最常说的权限控制原则

1.  Limitat  permissions for  instance credentials  权限最小化原则。
1. 确保cloud-instance-account仅具有必要的权限
1. 每个云提供商都应遵循一系列建议
1. 权限控制不在kubernetes中



![image.png](https://img-blog.csdnimg.cn/img_convert/75a92c3c7d7c9f0f2f664af2b22de6df.png#align=left&display=inline&height=546&margin=[objectObject]&name=image.png&originHeight=546&originWidth=985&size=127643&status=done&style=none&width=985)
## 3. restrict access using networkpolicies 使用网络策略限制访问
![image.png](https://img-blog.csdnimg.cn/img_convert/88b89d84465386dad0f83aaf4f1a47b5.png#align=left&display=inline&height=292&margin=[objectObject]&name=image.png&originHeight=584&originWidth=1124&size=218635&status=done&style=none&width=562)


### 3.1 限制访问云商的元数据
![image.png](https://img-blog.csdnimg.cn/img_convert/0b180e243bb7016a77d5da1bc59e9fbf.png#align=left&display=inline&height=530&margin=[objectObject]&name=image.png&originHeight=530&originWidth=1037&size=100777&status=done&style=none&width=1037)
由于没有谷歌云拿腾讯云意淫下了，可能理解的不是很对。往指教：
翻了下腾讯云的文档关于元数据也有文档：[https://cloud.tencent.com/document/product/213/4934?from=10680](https://cloud.tencent.com/document/product/213/4934?from=10680)
![image.png](https://img-blog.csdnimg.cn/img_convert/79443e619ba7e2da20d4eab49b23ee20.png#align=left&display=inline&height=86&margin=[objectObject]&name=image.png&originHeight=172&originWidth=1435&size=30892&status=done&style=none&width=717.5)
就简单的证明一下，node节点和pod节点都可以访问云商的源数据。相对于谷歌云的文档，腾讯的还是略简单，想比着课程查询下硬盘，貌似还是没有这接口的。不过觉得下面这话说的很对，能访问实例就可以查看元数据。关于元数据的安全也很重要啊......
![image.png](https://img-blog.csdnimg.cn/img_convert/fe7168ad9590b6a2caeca9d7c747bdc2.png#align=left&display=inline&height=102&margin=[objectObject]&name=image.png&originHeight=203&originWidth=760&size=18526&status=done&style=none&width=380)
### 3.2通过networkpolicy 限制对元数据的访问
ping metadata.tencentyun.com得到medata的地址169.254.0.23，依然是命名空间级别的限制
![image.png](https://img-blog.csdnimg.cn/img_convert/1a099467f440ca7bda540aba3aa94721.png#align=left&display=inline&height=194&margin=[objectObject]&name=image.png&originHeight=388&originWidth=701&size=21322&status=done&style=none&width=350.5)
```html
kubectl apply -f deny.yaml
kubectl -n metadata  exec nginx -it bash
curl http://metadata.tencentyun.com/latest/meta-data/instance/image-id
```
OK,如下图获取不了元数据中的镜像id了
![image.png](https://img-blog.csdnimg.cn/img_convert/23ca2a5341c80b8c5e953c2ee39631ec.png#align=left&display=inline&height=75&margin=[objectObject]&name=image.png&originHeight=149&originWidth=1267&size=22863&status=done&style=none&width=633.5)
然后我如何运行一组pod去访问元数据呢？
![image.png](https://img-blog.csdnimg.cn/img_convert/49efd8acdba7311695eb5a43e35765c6.png#align=left&display=inline&height=218&margin=[objectObject]&name=image.png&originHeight=436&originWidth=924&size=27125&status=done&style=none&width=462)
```html
kubectl apply -f allow.yaml
```
嗯 matchLabels我设置了一个不存在的，然后给pods加上labels
```html
kubectl get pods --show-labels -n metadata
kubectl label pod nginx role=metadata-accessor -n metadata
kubectl get pods --show-labels -n metadata
kubectl -n metadata  exec nginx -it bash
```
![image.png](https://img-blog.csdnimg.cn/img_convert/d6f19ac2d111cf12ec86addf7387278e.png#align=left&display=inline&height=202&margin=[objectObject]&name=image.png&originHeight=403&originWidth=1353&size=67371&status=done&style=none&width=676.5)
OK,可以返回元数据了。now现在去掉role=metadata-accessor 的标签
![image.png](https://img-blog.csdnimg.cn/img_convert/4eb04fefc1787831c9e8c17aeed97853.png#align=left&display=inline&height=86&margin=[objectObject]&name=image.png&originHeight=172&originWidth=1276&size=26426&status=done&style=none&width=638)
验证通过，其实我觉这节课主要的还是再强调networkpolicy。不仅仅是元数据的保护。networkpolicy是很重要的基石。




