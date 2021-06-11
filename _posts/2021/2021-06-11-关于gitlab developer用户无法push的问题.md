---
layout: post
title: 22021-06-11-关于gitlab developer用户无法push的问题
date: 2021-06-11 2:00:00
category: gitlab
tags: gitlab push
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

参见[Kubernetes 1.20.5 安装gitlab](https://cloud.tencent.com/developer/article/1808203)，搭建了gitlab也都是自己玩的，也没有添加什么新的用户。线上跑的有个老的8.5.8的版本貌似？一直也没有升级，跑了好些年了。昨天有个新的项目组要创建一个项目。so  group repository创建完成教了一下小伙伴的一般使用方式就跑路了。

今天早上group中Developer用户的小伙伴用小乌龟的客户端clone项目后add添加后无法push？

![image.png](/assets/images/2021/06-11/sudr90fxf6.png)

what?我特意试了一下。我的客户端是用的GitHub Desktop客户端。试着add push了一下 发现没有问题啊......

![image.png](/assets/images/2021/06-11/z6m9jhklj9.png)

看了下小伙伴的客户端上传的时候依然显示master分支，记得去年某些运动的时候 都改成main了啊 不会是这样的问题吧。尝试了一下排除......

# 解决问题：

## 1 . 解决gitlab developer用户无法push的问题

仔细研读了一下gitlab的权限设计，也仔细想了一下：developer怎么能把文件推送到master（main）分支呢？这本来就不应该是一个正常的方向。master（main）主分支的合并应该是master的权限！

鉴于大家都水开发，为了方便，百度了一下解决方案：

![image.png](/assets/images/2021/06-11/v0yxidgwb8.png)

是有好多这样的问题。但是我的gitlab版本是1.13.7来吧？貌似都有点不对头，依着葫芦画瓢找了下，总算找到了相关配置：

![image.png](/assets/images/2021/06-11/3q4vnacsd5.png)

让小伙伴试了下总算可以了......

# 总结一下：

## 1. gitlab or其他git项目管理方式都有完善开发方式，如git flow等。

## 2. 哎小公司还是普遍太水，仓库的使用和管理方式较为单一。并不能彰显出git的强大功能。

## 3. 个人也太水了。没有仔细研究过git的各种好的功能。

## 4. 希望git的强大功能能真正的用在工作中，而不是仅仅只作为一个所谓的仓库。

