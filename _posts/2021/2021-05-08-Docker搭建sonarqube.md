---
layout: post
title: Docker搭建sonarqube8.9
date: 2021-05-08 6:00:00
category: docker
tags: docker sonarqube8.9.0 lts
author: duiniwukenaihe
---
* content
{:toc}


# 前言：

 SonarQube 是一个用于代码质量管理的开源平台，用于管理源代码的质量。同时 SonarQube 还对大量的持续集成工具提供了接口支持，可以很方便地在持续集成中使用 SonarQube。此外 SonarQube 的插件还可以对 Java 以外的其他编程语言提供支持，对国际化以及报告文档化也有良好的支持。

# 特性:

- 多语言的平台： 支持超过20种编程语言，包括Java、Python、C#、C/C++、JavaScript等常用语言。
- 自定义规则： 用户可根据不同项目自定义Quality Profile以及Quality Gates。
- 丰富的插件： SonarQube 拥有丰富的插件，从而拥有强大的可扩展性。
- 持续集成： 通过对某项目的持续扫描，可以对该项目的代码质量做长期的把控，并且预防新增代码中的不严谨和冗余。
- 质量门： 在扫描代码后可以通过对“质量门”的比对判定此次“构建”的结果是否通过，质量门可以由用户定义，由多维度判定是否通过。

注：这东西个人还是仅测试不敢玩哈哈哈。个性的程序员太多，出现各种各样的坏味道对应小运维来说也不知道怎么该跟程序所解释对接。不能作为一个清晰既定的衡量标准去衡量各种。各种阀值做不到正确的配置。只用于演示演示。

# 关于SonarQube 的版本

1. Community
2. Developer
3. Enterprise
4. Data Center

过去个人玩是只在kubernetes上部署了SonarQube 的7.9lts版本（community社区版本）。lts顾名思义长期稳定支持版。kubernetes的部署方式可以参见豆丁大佬的博文：[http://www.mydlq.club/article/25/](http://www.mydlq.club/article/25/)。个人也写个一篇类似的文章helm的安装方式搭建：[https://duiniwukenaihe.github.io/2019/11/29/k8s-helm-install-postgresql-sonarqube/](https://duiniwukenaihe.github.io/2019/11/29/k8s-helm-install-postgresql-sonarqube/)

# 一. SonarQube部署过程：

注：报名了泽阳大佬的jenkins CI/CD训练营。基本就是按照阳明大佬的步骤来的.想更深入学习的可以报名：[https://www.idevops.site/](https://www.idevops.site/)。物超所值。满满的干货。当然了大佬课程是搭建的7.9.6的版本，我是直接玩8.9.0的lts了。

## 1. 前提条件：

个人环境是在内网搭建的proxmox虚拟化的服务器上虚拟的centos8.3.并参照[https://blog.csdn.net/qq\_41570843/article/details/106182073](https://blog.csdn.net/qq_41570843/article/details/106182073)安装了docker环境的初始化主机。

主机ip为：192.168.0.109

## 2. docker搭建SonarQube

关于 lts镜像 为什么用sonarqube:8.9.0-community image呢？可以到dockerhub看下两个镜像的tag

![image.png](/assets/images/2021/05-08/zcphdmix3h.png)

```
## 创建数据目录
mkdir -p /data/sonarqube/{sonarqube_conf,sonarqube_extensions,sonarqube_logs,sonarqube_data}
chmod 777 -R /data/sonarqube/

## 运行
docker run  -itd  --name sonarqube \
    -p 9000:9000 \
    -v /data/sonarqube/sonarqube_conf:/opt/sonarqube/conf \
    -v /data/sonarqube/sonarqube_extensions:/opt/sonarqube/extensions \
    -v /data/sonarqube/sonarqube_logs:/opt/sonarqube/logs \
    -v /data/sonarqube/sonarqube_data:/opt/sonarqube/data \
    sonarqube:8.9.0-community
## 验证
docker logs -f sonarqube
```

登陆web管理页面初始用户名密码dmin修改密码。7.9的版本应该是没有默认修改密码的这一步的，会直接登陆控制台页面。初始化修改密码这步在安全性上我个人觉得这也是一个进步。

![image.png](/assets/images/2021/05-08/erjllxvj39.png)

## 3. 在线安装插件（中文插件为例）

Adminstration-Marketplace-Plugins

先提示一条风险l understand the risk

我了解风险。（这一步应该也是新的版本增加的7.9.6的lts貌似是没有的）然后install 搜索chinese install插件，然后重启。（网络经常抽筋安装不了）

![image.png](/assets/images/2021/05-08/hn52xh1pel.png)

![image.png](/assets/images/2021/05-08/t286cvnopv.png)

## 4. 手动下载插件离线安装

[https://github.com/xuhuisheng/sonar-l10n-zh/tree/sonar-l10n-zh-plugin-8.8#the-chinese-translation-pack-for-sonarqube](https://github.com/xuhuisheng/sonar-l10n-zh/tree/sonar-l10n-zh-plugin-8.8#the-chinese-translation-pack-for-sonarqube)查看版本对应关系

[https://github.com/xuhuisheng/sonar-l10n-zh/releases/tag/sonar-l10n-zh-plugin-8.9](https://github.com/xuhuisheng/sonar-l10n-zh/releases/tag/sonar-l10n-zh-plugin-8.9)

![image.png](/assets/images/2021/05-08/2f6nb92ejl.png)

![image.png](/assets/images/2021/05-08/1tohwxe9zg.png)

上传到/data/sonarqube/sonarqube\_extensions/downloads目录，然后

```
cd /data/sonarqube/sonarqube_extensions/downloads
wget https://github.com/xuhuisheng/sonar-l10n-zh/releases/download/sonar-l10n-zh-plugin-8.9/sonar-l10n-zh-plugin-8.9.jar
chmod +x sonar-l10n-zh-plugin-1.29.jar
docker restart sonarqube
```

## 5.配置强制登录

![image.png](/assets/images/2021/05-08/sh4ig7t2p8.png)

![image.png](/assets/images/2021/05-08/f47jril3n2.png)

## 6. 将容器中lib目录复制到本地，并在容器中挂载本地目录

其实是加深下docker cp的用法了

```
## lib目录

mkdir -p /data/sonarqube/sonarqube_lib
cd  /data/sonarqube/sonarqube_lib
docker cp sonarqube://opt/sonarqube/lib/. /data/sonarqube/sonarqube_lib/
## 销毁旧的容器，重新挂载本地目录
docker stop sonarqube
docker rm sonarqube
docker run  -itd  --name sonarqube \
    -p 9000:9000 \
    -v /data/sonarqube/sonarqube_conf:/opt/sonarqube/conf \
    -v /data/sonarqube/sonarqube_extensions:/opt/sonarqube/extensions \
    -v /data/sonarqube/sonarqube_logs:/opt/sonarqube/logs \
    -v /data/sonarqube/sonarqube_data:/opt/sonarqube/data \
    -v /data/sonarqube/sonarqube_lib:/opt/sonarqube/lib \
    sonarqube:8.9.0-community
```

在lib目录下新建一个文件，登陆容器挂载目录内验证加载成功

![image.png](/assets/images/2021/05-08/1e3q9eznmr.png)

注：这一步后web登陆sonarqube应该又要重新初始化一遍密码。因为数据库没有整外部的也没有挂载数据目录。使用的默认的数据库内部的。生产环境应该起码配置外部的数据库的嗯大部分用的是postgresql…

## 7. 关于插件的版本与对应关系

在sonarqube7.9版本中 常用的插件举个例子：

1. java -Java Code Quality and Security
2. js-SonarJS
3. GO-SonarGo

8.9版本中很多不需要安装了....参照：[https://docs.sonarqube.org/latest/instance-administration/plugin-version-matrix/](https://docs.sonarqube.org/latest/instance-administration/plugin-version-matrix/)。凡是提示Bundled的都已经默认集成了：

![image.png](/assets/images/2021/05-08/myu8b6fv4e.png)

# 二. 配置Scanner

## 1. 关于Scanner:

> Gradle - [SonarScanner for Gradle](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-gradle/)
> .NET - [SonarScanner for .NET](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/)
> Maven - use the [SonarScanner for Maven](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-maven/)
> Jenkins - [SonarScanner for Jenkins](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-jenkins/)
> Azure DevOps - [SonarQube Extension for Azure DevOps](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-azure-devops/)
> Ant - [SonarScanner for Ant](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-ant/)
> anything else (CLI) - [SonarScanner](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/)

选择anything else了，cli的方式。直接在192.168.0.173这台搭建了gitlab的主机上面搭建了（这台主机我也搞了jenkins slave。安装了jdk的1.8）

关于主机已经运行的gitlab环境参照：[https://cloud.tencent.com/developer/article/1818427](https://cloud.tencent.com/developer/article/1818427)

## 2. 关于版本：

[https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/](https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/)一般是安装最新的我个人。但是下载的时候貌似没有看好...选择了sonar-scanner-cli-4.6.1.2450-linux.zip。反正大同小异的就拿这个来演示了......

![image.png](/assets/images/2021/05-08/71bgjlv24p.png)

## 3. 安装scanner

```
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.6.1.2450-linux.zip
unzip sonar-scanner-cli-4.6.1.2450-linux.zip
mv sonar-scanner-4.6.1.2450-linux /usr/local/
```

vim /etc/profile

```
export SCANNER_HOME=/usr/local/sonar-scanner-4.6.1.2450-linux
export PATH=$PATH:$SCANNER_HOME/bin
```

source/etc/profile

确认版本安装成功生效

```
sonar-scanner -v
```

what嗯  我本地的环境是jdk1.8d  为什么我的sonar-scanner java版本是java11呢？这是因为sonar-scanner中集成了jre环境。默认是使用自己的jre的.

![image.png](/assets/images/2021/05-08/2cpqb0nyci.png)

## 4. 修改scanner的java使用版本

```
vim /usr/local/sonar-scanner-4.6.1.2450-linux/bin/sonar-scanner
###修改use_embedded_jre参数
use_embedded_jre=false
```

![image.png](/assets/images/2021/05-08/60e4awzgze.png)

![image.png](/assets/images/2021/05-08/ez9e2ss05k.png)

sonar-scanner -v 验证java版本

![image.png](/assets/images/2021/05-08/ipmnyd2dzq.png)

# 三. SonarScanner的简单使用

只是简单验证使用下sonnarscanner的使用

## 1. maven的安装

注：其实在安装jenkins的时候已经安装了maven了。但是这里没有写jenkins的部署。所以这里就补写一下了。

```
### 下载

wget  https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
tar zxf apache-maven-3.8.1-bin.tar.gz -C /usr/local/
cd /usr/local/apache-maven-3.8.1/
pwd  /usr/local/apache-maven-3.8.1


### 配置环境变量
vi /etc/profile
export M2_HOME=/usr/local/apache-maven-3.8.1
export PATH=$M2_HOME/bin:$PATH
source /etc/profile


### 验证

mvn -v
Apache Maven 3.8.1 (05c21c65bdfed0f71a2f2ada8b84da59348c4c5d)
Maven home: /usr/local/apache-maven-3.8.1
Java version: 1.8.0_292, vendor: Red Hat, Inc., runtime: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.292.b10-0.el8_3.x86_64/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "4.18.0-240.22.1.el8_3.x86_64", arch: "amd64", family: "unix"
```

## 2.初始化一个maven项目

### 1. 初始化一个springboot  maven项目

\*\* \*\*[https://start.spring.io/](https://start.spring.io/)的maven项目

![](/assets/images/2021/05-08/xgmcz9wrer.png)

### 2. 上传到gitlab私有仓库

其实是跟着泽阳大佬的步骤做的一个devops-maven-service gitlab项目。反正就是那么的一个demo没有做什么复杂的设置。其实如果只是只sonarqube集成也可以不先整合到gitlab的，但是为了尽量还原自己的环境，故上传到gitlab了。

![image.png](/assets/images/2021/05-08/p651d08unt.png)

### 3. 克隆仓库

```
git clone http://192.168.0.173/devops/devops-maven-service
```

### 4. maven打包

```
mvn clean package
```

一定记得mvn打包。自己脑袋糊涂了。不maven打包哪里会有target class目录呢......

执行扫描会报错的（扫描代码详见下一步）

![image.png](/assets/images/2021/05-08/nrxbkojmlr.png)

![image.png](/assets/images/2021/05-08/atdarrehtj.png)

### 5. sonar扫描

```
    sonar-scanner -Dsonar.host.url=http://192.168.0.209:9000 \
-Dsonar.projectKey=devops-maven-service \
-Dsonar.projectName=devops-maven-service \
-Dsonar.projectVersion=1.0 \
-Dsonar.login=admin \
-Dsonar.password=abc@1234 \
-Dsonar.ws.timeout=30 \
-Dsonar.projectDescription="my first project!" \
-Dsonar.links.homepage=http://192.168.0.173/devops/devops-maven-service \
-Dsonar.links.ci=http://192.168.0.205:8080/job/demo-pipeline-service/ \
-Dsonar.sources=src \
-Dsonar.sourceEncoding=UTF-8 \
-Dsonar.java.binaries=target/classes \
-Dsonar.java.test.binaries=target/test-classes \
-Dsonar.java.surefire.report=target/surefire-reports
```

嗯配置参数基本字面意思homepage ci都是乱写的可以忽略。

具体参数实例可以参考： [https://docs.sonarqube.org/latest/analysis/analysis-parameters/](https://docs.sonarqube.org/latest/analysis/analysis-parameters/)

各种语言的扫描示例：[https://docs.sonarqube.org/latest/analysis/languages/](https://docs.sonarqube.org/latest/analysis/languages/go/)

反正正常扫描完了就是下面专业的

![image.png](/assets/images/2021/05-08/sqyll5nr87.png)

### 6. 登陆sonarqube web查看结果

介绍后登陆sonarqubeweb管理页面查看：

![image.png](/assets/images/2021/05-08/ljsztl2k1w.png)

### 7.就想拿一个自己的项目跑一下

嗯 貌似就是这样的。拿自己内部的一个tts的项目测试下（起名是个学问，这样的名字我也很无语。其实就是一个使用阿里云tts文字转语音的服务）。

#### 1. 下载代码到sonar服务器忽略

#### 2. mvn 打包

```
mvn clean package
```

讲真出现那么多Positive matches 我有点强迫症。都是大佬写的。就抛砖引玉了......

![image.png](/assets/images/2021/05-08/cgwhbx9j74.png)

#### 3. sonar扫描

```
    sonar-scanner -Dsonar.host.url=http://192.168.0.209:9000 \
-Dsonar.projectKey=tts \
-Dsonar.projectName=tts \
-Dsonar.projectVersion=1.0 \
-Dsonar.login=admin \
-Dsonar.password=abc@1234 \
-Dsonar.ws.timeout=30 \
-Dsonar.projectDescription="my first project!" \
-Dsonar.links.homepage=http://192.168.0.173/devops/devops-maven-service \
-Dsonar.links.ci=http://192.168.0.205:8080/job/demo-pipeline-service/ \
-Dsonar.sources=src \
-Dsonar.sourceEncoding=UTF-8 \
-Dsonar.java.binaries=target/classes \
-Dsonar.java.test.binaries=target/test-classes \
-Dsonar.java.surefire.report=target/surefire-reports
```

#### 4. 登陆sonarqube查看

扫描完成登陆sonarqube查看 嗯 tts的应用也有了

![image.png](/assets/images/2021/05-08/mrf08jyawx.png)

# 后记：

对于我来说和鬼东西段时间还是用不起来。只能算是扩展下自己的知识面，了解下人家的思想和流程......。然后再准备搞一下与jenkins的流程。就为了体验一下正常的cicd工具流过程。