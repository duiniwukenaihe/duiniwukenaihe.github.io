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
**svn sqlite[S13]:database or disk is full**
# 2. 解决问题过程：
## 1. 扩容svn服务器硬盘
没有搞明白怎么回事....怎么还有sqlite?是不是我的硬盘小了呢？看了下我分配的svn硬盘是250G。我的总文件也就120G理论上来说不应该的。抱着司马当活马医的态度。我扩容了硬盘到了400G，嗯还是出现了这个问题。这样也让我确认应该不是硬盘的问题了。
## 2.删除项目重新上传。观察svn服务器
然后我就把repo下这个svn项目删除了。重新建了项目然后重新add commit两个小的文件夹上传观察：
svn服务下项目文件夹下有一下几个目录
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619584940746-2932e182-430c-421d-8010-272b9b5f0dae.png#clientId=uc212b2cf-d3d4-4&from=paste&height=141&id=u7444cd76&margin=%5Bobject%20Object%5D&name=image.png&originHeight=141&originWidth=679&originalType=binary&size=14876&status=done&style=none&taskId=u287b1e4b-eecd-4ba5-b60e-d624b169d15&width=679)
看了一眼db目录是最大的。而且个人理解应该也确实在db目录下的
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619585021330-07d99590-9b2f-4203-a675-66918941a784.png#clientId=uc212b2cf-d3d4-4&from=paste&height=256&id=u1ff8f981&margin=%5Bobject%20Object%5D&name=image.png&originHeight=256&originWidth=758&originalType=binary&size=31535&status=done&style=none&taskId=u5386253e-d953-457f-a51c-c8633b96243&width=758)
版本1 提交500M文件夹的时候发现上传过程中txn-protorevs目录不断增加。上传完了以后txn-protorevs目录就空了，染然后revs目录下就有了一个0的目录。
然后上传20G文件目录试试：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619585145767-6564ccbf-4d93-490c-b83b-f0d940e33b45.png#clientId=uc212b2cf-d3d4-4&from=paste&height=718&id=u9dec45fd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=718&originWidth=727&originalType=binary&size=62317&status=done&style=none&taskId=u09f48656-8da6-436a-b0c7-6aff4a66db0&width=727)
然后这样的话个人就基本能够明白了：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619585253341-bd06b3a3-b74c-4970-bae6-1b5cb8898bae.png#clientId=uc212b2cf-d3d4-4&from=paste&height=183&id=u4fb0e1e7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=183&originWidth=630&originalType=binary&size=18549&status=done&style=none&taskId=ufc627752-ca81-47ad-a3d2-7250c585cf2&width=630)
svn在上传的过程中再txn-protorevs目录下生成对应版本tag的rev rev-lock文件。等待上传完成在revs目录下生成对应版本的文件.
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619585371021-f3bfee9b-60ac-4124-bceb-bdf824a06b27.png#clientId=uc212b2cf-d3d4-4&from=paste&height=151&id=ua0a3fe26&margin=%5Bobject%20Object%5D&name=image.png&originHeight=151&originWidth=829&originalType=binary&size=15563&status=done&style=none&taskId=u616a1f2a-bd27-47e4-aedb-f3462572d14&width=829)
# 3. 后知后觉
既然出现了sqlite那我是不是可以理解txn-protorevs目录下 rev 文件是一个sqlite文件呢？这个文件的大小超出了限制呢？
我把大文件夹拆分成四次add commit 上传.......最后总算成功了。通过百度或者Google没有能获取svn 的sqlite临时保存的这个rev文件的最大大小是多。不知道有没有大佬能够帮忙解惑解答一下呢。20G的资源上传我是没有出什么问题的。大于50G是出现了问题。具体的最大文件大小也没有能确认。反正就是尽量不一次性上传太大的文件了。
