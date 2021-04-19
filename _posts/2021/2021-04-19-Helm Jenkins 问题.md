---
layout: post
title: Helm Jenkins 安装后时区问题的强迫症
date: 2021-04-19 04:00:00
category: Jenkins
tags: Helm kubernetes
author: duiniwukenaihe
---
* content
{:toc}

helm 安装了jenkins构建应用的使用不知道有没有跟我出现同样问题的小伙伴：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618821079370-866bda1e-b4cf-4a22-8c10-71528b53637e.png#clientId=ub75c302e-4e8f-4&from=paste&height=717&id=u298c680f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=717&originWidth=1530&originalType=binary&size=101166&status=done&style=none&taskId=uc43a00bf-60fe-4ec4-9c48-2e3c879f073&width=1530)
build history中的时区timezone是utc  但是阶段视图的是正常的东八区上海时区....what....我改怎么办？个人有点强迫症。helm玩的不太好。看了一眼官方文档也没有找到修改时区的方法？难道要我去重新制作镜像 修改时区？讲真我有点后悔用helm了......
无意中google找到了一个插件[https://chrome.google.com/webstore/detail/jenkins-local-timezone/omjcneepammbodkobeobihfpfngbdjoc](https://chrome.google.com/webstore/detail/jenkins-local-timezone/omjcneepammbodkobeobihfpfngbdjoc)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618821229457-758bff38-3162-4dad-a3fb-a6c678ee93ae.png#clientId=ub75c302e-4e8f-4&from=paste&height=958&id=ua28d682e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=958&originWidth=1556&originalType=binary&size=227772&status=done&style=none&taskId=u36062f5a-e948-4379-9d30-f8a30149578&width=1556)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618821245035-7ed428e0-81d3-4e30-aa46-48c3cbed3848.png#clientId=ub75c302e-4e8f-4&from=paste&height=399&id=ue1188805&margin=%5Bobject%20Object%5D&name=image.png&originHeight=399&originWidth=1322&originalType=binary&size=57055&status=done&style=none&taskId=u9ad4f763-1b16-4314-ac03-240ff24503f&width=1322)
安装插件如下：自己骗一下自己吧......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618821334574-7a164c98-972c-4b2f-ad15-4c67deb64a5c.png#clientId=ub75c302e-4e8f-4&from=paste&height=749&id=ub1f6f3df&margin=%5Bobject%20Object%5D&name=image.png&originHeight=749&originWidth=1584&originalType=binary&size=109115&status=done&style=none&taskId=u4697294d-8d6b-412f-95b7-ab80fef0d27&width=1584)
以后要么有时间好好研究一下helm。要么还是yaml  deployment的方式去安装吧。尽量选择自己擅长的方式。