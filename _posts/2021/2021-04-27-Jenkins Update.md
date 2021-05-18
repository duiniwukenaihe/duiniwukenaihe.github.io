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
2. 定期发布 (每周)

默认反正我个人是tls稳定版了。

![image.png](/assets/images/2021/04-27/vwqvs5aea9.png)

我tomcat的目录在/data/wwwroot/default。 将jar包加压到目录下（war包可以直接解压的）。重新启动tomcat服务：

```
service tomcat start
```

## 4. 验证更新后服务是否正常启动

web访问http://192.168.0.170

![image.png](/assets/images/2021/04-27/natffx7k3c.png)

what？出现异常了。不能访问......。看了一眼tomcat的日志：

![image.png](/assets/images/2021/04-27/5ks2lldspr.png)

更新到最新版本以失败告终........，log中截取了报错信息如下：

```
enkins.model.Jenkins.save An attempt to save Jenkins'' global configuration before it has been loaded has been made during milestone Configuration for all jobs updated.  This is indicative of a bug in the caller and may lead to full or partial loss of configuration
```

# 2. 复盘更新

## 1. 万能的百度orgoogle

将log中报错复制到了百度搜索，找到了stackoverflow中的一个类似的更新失败案例：

[https://stackoverflow.com/questions/65441139/how-to-fix-jenkins-java-lang-illegalstateexception-an-attempt-to-save-the-globa](https://stackoverflow.com/questions/65441139/how-to-fix-jenkins-java-lang-illegalstateexception-an-attempt-to-save-the-globa)。得到一下两个方案：

### 1. 降级：

![image.png](/assets/images/2021/04-27/8smt28h8oj.png)

### 2. 更新[role-strategy-3.1](https://github.com/jenkinsci/role-strategy-plugin/releases/tag/role-strategy-3.1).hpi

![image.png](/assets/images/2021/04-27/opo6fho97l.png)

按照这里面的方案试一下了......

## 2. 将jenkins 2.235.3更新到2.263版本

重复1.1-1.4过程。当然了 只是变成了下载war包下载2.263.1的war包。嗯更新完可以正常进入：

任务就忽略了，瞎写着测试的......

![image.png](/assets/images/2021/04-27/180t49hr2h.png)

接着把版本升级到1.263.4的版本看一眼，嗯 也成功了

![image.png](/assets/images/2021/04-27/6dz567j2jl.png)

这个时候如果想抱着直接升级到2.277.3就能成功的侥幸还是打错特错的.....

## 3. jenkins2.263版本安装插件

仔细观察我的jenkins中右上角提示信息里面有那么一堆，看了一眼

-   Icon Shim Plugin
-   jQuery UI plugin
-  Pipeline: Declarative Agent API

这几个插件我也是没有用的。还有一些插件也都过时了....我就把能看到的这些插件都卸载了......

![image.png](/assets/images/2021/04-27/my4ha0ozo9.png)

![image.png](/assets/images/2021/04-27/o17dl57be2.png)

其他插件类似....不相干的插件还是少安装了......

然后下载role-straegy.hpi并上传到jenkins:

系统管理-插件管理-高级-上传插件。整完以后重启tomcat

![image.png](/assets/images/2021/04-27/d2swh6wm5t.png)

## 4. 继续更新jenkins

重新执行1.1-1.4流程，嗯版本总算更新成功了

![image.png](/assets/images/2021/04-27/oaj098du84.png)

查看tomcat log:

![image.png](/assets/images/2021/04-27/5stm2fjei5.png)

# 总结：

通过这次更新个人的总结：

1. 更新升级前要做好程序与配置文件的备份（比较有些配置文件与程序是分开的）
2. 深入了解一下版本的更新文档？jenkins在1.277版本应该就是做了什么的更改的。1.235-1.263是可以直接升级的。
3. 尽量少安装不必要的插件。以免引起版本更新过程中的不兼容问题。
4. 善于查看日志并用各种搜索工具......




