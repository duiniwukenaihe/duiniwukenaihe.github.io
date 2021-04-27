---
layout: post
title: Jenkins Update
date: 2021-04-27 6:00:00
category: Jenkins
tags: centos jenkins
author: duiniwukenaihe
---
* content
{:toc}

# 背景
内网有一台项目组用的jenkins，ip 192.168.0.170.版本为1.235.3的版本。部署方式为 tomcat war包+nginx代理。正好有时间想把jenkins升级到最新版本。说干就干，下面记录一下升级的痛苦过程......
# 1. jenkins升级的痛苦过程
按照官方的文档也一般的安装过程就是下载最新jar包替换这样的流程。故：


## 1. 停止 jenkins服务


```
####远古项目centos7还是默认的service
service tomcat stop
###centos 8 可以systemctl
systemctl stop tomcat
```
当然了 如果是jar的方式启动的 可以kill了 jar进程。
## 2. 备份jenkins服务
先备份jenkins数据目录文件和版本部署文件
我的集成环境使用的oneinstck安装的java nginx环境。配置文件在/home/www./jenkins目录下
jar包部署在/data/wwwroot/default目录。先做个备份
```
####
cd /home/www/
cp -Ra .jenkins/ .jenkinsold/
cd /data/wwwroot
cp -Ra default defaultold
```
## 3. 下载最新war包更新到服务器并启动服务
[https://get.jenkins.io/war-stable/](https://get.jenkins.io/war-stable/)下载了最新的2.277.3版本，关于 lts版本与weekly版本:

1. 稳定版 (LTS)
1. 定期发布 (每周)

默认反正我个人是tls稳定版了。




![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619497202008-d74a4ed5-029e-479d-b630-6c9cd9a16931.png#clientId=udb3a234d-1c09-4&from=paste&height=468&id=u43a7c0d6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=468&originWidth=1286&originalType=binary&size=44046&status=done&style=none&taskId=uf49e8dce-67ee-4a1a-9328-4e700d6460b&width=1286)
我tomcat的目录在/data/wwwroot/default。 将jar包加压到目录下（war包可以直接解压的）。重新启动tomcat服务：
```
service tomcat start
```
## 4. 验证更新后服务是否正常启动
web访问http://192.168.0.170
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619505821713-2a57b652-c185-4fc4-aef0-74b2d240d6b6.png#clientId=udb3a234d-1c09-4&from=paste&height=719&id=u3100d189&margin=%5Bobject%20Object%5D&name=image.png&originHeight=719&originWidth=1272&originalType=binary&size=70134&status=done&style=none&taskId=uc8d4c6d5-0f06-4192-8835-ad41dff4d28&width=1272)
what？出现异常了。不能访问......。看了一眼tomcat的日志：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619505865460-e9bc35e5-fd75-44a8-93f1-c885781c2bc9.png#clientId=udb3a234d-1c09-4&from=paste&height=671&id=u70f40c6e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=671&originWidth=1518&originalType=binary&size=141043&status=done&style=none&taskId=u88b7a729-31f0-4b1d-b560-1896b4b323e&width=1518)
更新到最新版本以失败告终........，log中截取了报错信息如下：
```
enkins.model.Jenkins.save An attempt to save Jenkins'' global configuration before it has been loaded has been made during milestone Configuration for all jobs updated.  This is indicative of a bug in the caller and may lead to full or partial loss of configuration
```
# 2. 复盘更新
## 1. 万能的百度orgoogle
将log中报错复制到了百度搜索，找到了stackoverflow中的一个类似的更新失败案例：
[https://stackoverflow.com/questions/65441139/how-to-fix-jenkins-java-lang-illegalstateexception-an-attempt-to-save-the-globa](https://stackoverflow.com/questions/65441139/how-to-fix-jenkins-java-lang-illegalstateexception-an-attempt-to-save-the-globa)。得到一下两个方案：
### 1. 降级：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619506213835-13e85984-9190-48ac-8023-e4d89c5066c2.png#clientId=udb3a234d-1c09-4&from=paste&height=247&id=u46ba7969&margin=%5Bobject%20Object%5D&name=image.png&originHeight=247&originWidth=868&originalType=binary&size=21903&status=done&style=none&taskId=ude2169af-bffd-48dd-a977-873a8a8fb91&width=868)
### 2. 更新[role-strategy-3.1](https://github.com/jenkinsci/role-strategy-plugin/releases/tag/role-strategy-3.1).hpi
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619506239562-81b69685-2eb8-46fa-a516-0030a0d5fe1d.png#clientId=udb3a234d-1c09-4&from=paste&height=420&id=ufa991067&margin=%5Bobject%20Object%5D&name=image.png&originHeight=420&originWidth=1099&originalType=binary&size=36415&status=done&style=none&taskId=u9892f2c2-b6f5-4b18-8b25-05cf0f242b2&width=1099)
按照这里面的方案试一下了......
## 2. 将jenkins 2.235.3更新到2.263版本
重复1.1-1.4过程。当然了 只是变成了下载war包下载2.263.1的war包。嗯更新完可以正常进入：
任务就忽略了，瞎写着测试的......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619506542215-1faf8b3e-7814-4dd9-8d5b-fe505d1f8a99.png#clientId=udb3a234d-1c09-4&from=paste&height=755&id=ub61ae7d3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=755&originWidth=1249&originalType=binary&size=197566&status=done&style=none&taskId=uc4815ce4-d1a6-4910-b513-b7658ceff55&width=1249)
接着把版本升级到1.263.4的版本看一眼，嗯 也成功了
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619506506883-6101e47a-f642-426d-8e1d-39e7162bebcb.png#clientId=udb3a234d-1c09-4&from=paste&height=652&id=u530a236d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=652&originWidth=1261&originalType=binary&size=175129&status=done&style=none&taskId=ua19951dd-c523-4b0b-9705-f3e438aa443&width=1261)
这个时候如果想抱着直接升级到2.277.3就能成功的侥幸还是打错特错的.....
## 3. jenkins2.263版本安装插件
仔细观察我的jenkins中右上角提示信息里面有那么一堆，看了一眼

-       Icon Shim Plugin
-       jQuery UI plugin
-      Pipeline: Declarative Agent API

这几个插件我也是没有用的。还有一些插件也都过时了....我就把能看到的这些插件都卸载了......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619506784231-eb9c958a-4590-4c31-be95-f50d475bbc74.png#clientId=udb3a234d-1c09-4&from=paste&height=891&id=u67e86668&margin=%5Bobject%20Object%5D&name=image.png&originHeight=891&originWidth=1628&originalType=binary&size=125606&status=done&style=none&taskId=u867486ec-6b6e-4777-8361-9fd697187ea&width=1628)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619507099490-b4cd5b6d-4119-4695-94c8-7e65d994461c.png#clientId=udb3a234d-1c09-4&from=paste&height=622&id=u6cc1e57e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=622&originWidth=1460&originalType=binary&size=67241&status=done&style=none&taskId=uc2d5e971-0650-4151-ade4-c20e136615c&width=1460)
其他插件类似....不相干的插件还是少安装了......
然后下载role-straegy.hpi并上传到jenkins:
系统管理-插件管理-高级-上传插件。整完以后重启tomcat
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619507163565-c0d966fa-9257-4948-a69a-afdc34cbfe25.png#clientId=udb3a234d-1c09-4&from=paste&height=631&id=u4dcc98b2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=631&originWidth=1727&originalType=binary&size=161760&status=done&style=none&taskId=ue70f993b-f09b-48a6-8817-bb123d93a38&width=1727)
## 4. 继续更新jenkins
重新执行1.1-1.4流程，嗯版本总算更新成功了
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619507289210-52b9ac0a-34d0-466b-8fbe-b456f88862e8.png#clientId=udb3a234d-1c09-4&from=paste&height=820&id=u920d312b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=820&originWidth=1645&originalType=binary&size=99897&status=done&style=none&taskId=u5f162cdf-7b48-45dc-843a-c77c45d1359&width=1645)
查看tomcat log:
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619507316359-e6f22043-f486-4ee9-b896-a23c80b65306.png#clientId=udb3a234d-1c09-4&from=paste&height=610&id=u0ab9c8b9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=610&originWidth=1610&originalType=binary&size=170704&status=done&style=none&taskId=uc0a205dc-7286-480f-9949-36b8d053cd3&width=1610)
# 总结：
通过这次更新个人的总结：

1. 更新升级前要做好程序与配置文件的备份（比较有些配置文件与程序是分开的）
1. 深入了解一下版本的更新文档？jenkins在1.277版本应该就是做了什么的更改的。1.235-1.263是可以直接升级的。
1. 尽量少安装不必要的插件。以免引起版本更新过程中的不兼容问题。
1. 善于查看日志并用各种搜索工具......





