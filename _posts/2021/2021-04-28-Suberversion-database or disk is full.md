---
layout: post
title: Suberversion-database or disk is full
date: 2021-04-28 6:00:00
category: SVN
tags: svn sqlite
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

历史遗留问题。公司有一个古老的共享文件系统。权限的管理比较混轮也没有版本的管理。跟总办的同事商量了一下给他们迁移到了svn。文件大概有100多个G吧。搭建了一个svnmanager管理的svn系统。至于不用git或者其他的版本管理系统....是因为我觉得svn对于他们来说容易操作吧。

# 1. 出现的问题：

开始文件夹目录有三个，其中两个的文件夹一个20G一个 500多M，另外一个文件夹目录有80G左右。我就三个目录分别add commit了。20 G 和500M的两个文件夹很容易就加入了svn了。80多g的项目等待上传了一个晚上。早上到公司一看报错......   

**svn sqliteS13:database or disk is full**

# 2. 解决问题过程：

## 1. 扩容svn服务器硬盘

没有搞明白怎么回事....怎么还有sqlite?是不是我的硬盘小了呢？看了下我分配的svn硬盘是250G。我的总文件也就120G理论上来说不应该的。抱着司马当活马医的态度。我扩容了硬盘到了400G，嗯还是出现了这个问题。这样也让我确认应该不是硬盘的问题了。

## 2.删除项目重新上传。观察svn服务器

然后我就把repo下这个svn项目删除了。重新建了项目然后重新add commit两个小的文件夹上传观察：

svn服务下项目文件夹下有一下几个目录

![image.png](/assets/images/2021/04-28/heowo303f8.png)

看了一眼db目录是最大的。而且个人理解应该也确实在db目录下的

![image.png](/assets/images/2021/04-28/1yvp91rrg2.png)

版本1 提交500M文件夹的时候发现上传过程中txn-protorevs目录不断增加。上传完了以后txn-protorevs目录就空了，染然后revs目录下就有了一个0的目录。

然后上传20G文件目录试试：

![image.png](/assets/images/2021/04-28/dvm5ymxegh.png)

然后这样的话个人就基本能够明白了：

![image.png](/assets/images/2021/04-28/mr5fo4r0za.png)

svn在上传的过程中再txn-protorevs目录下生成对应版本tag的rev rev-lock文件。等待上传完成在revs目录下生成对应版本的文件.

![image.png](/assets/images/2021/04-28/li2vx8eb2n.png)

# 3. 后知后觉

既然出现了sqlite那我是不是可以理解txn-protorevs目录下 rev 文件是一个sqlite文件呢？这个文件的大小超出了限制呢？

我把大文件夹拆分成四次add commit 上传.......最后总算成功了。通过百度或者Google没有能获取svn 的sqlite临时保存的这个rev文件的最大大小是多。不知道有没有大佬能够帮忙解惑解答一下呢。20G的资源上传我是没有出什么问题的。大于50G是出现了问题。具体的最大文件大小也没有能确认。反正就是尽量不一次性上传太大的文件了。