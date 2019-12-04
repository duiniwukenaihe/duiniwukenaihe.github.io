---
layout: post
title: "2019-12-03-k8s-jenkins-sonarqube"
date: "2019-12-03 10:00:00"
category: "kubernetes"
tags:  "kubernetes  jenkins sonarqube"
author: duiniwukenaihe
---
* content
{:toc}

集群配置：
初始集群环境kubeadm 1.16.3

|  ip           | 自定义域名         |    主机名 |
|  :----:       |     :----:        |   :----:  |
|192.168.3.8      |  master.k8s.io    |  k8s-vip  |
|192.168.3.10    |  master01.k8s.io  |  k8s-master-01|
|192.168.3.5   |  master02.k8s.io  |  k8s-master-02| 
|192.168.3.12   |  master03.k8s.io  |  k8s-master-03|
|192.168.3.6    |  node01.k8s.io    |  k8s-node-01|
|192.168.3.2    |  node02.k8s.io    |  k8s-node-02|
|192.168.3.4    |  node03.k8s.io    |  k8s-node-03|

# 描述背景：
> 示例过程参考http://www.mydlq.club/article/11/超级小豆丁文档进行操作，版本过程略有不同。初衷是没有进过大公司，羡慕大公司的工作流，安装下sonarqube与jenkins集成跑个测试用例自己安慰下自己......
上一张抄来的流程图

![jenkins-sonar1](/assets/images/sonar/jenkins-sonar1.png)

# sonarqube配置

## 1. 禁用SCM传感器
> 点击 配置—SCM—Disable the SCM Sensor 将其关闭
![jenkins-sonar2](/assets/images/sonar/jenkins-sonar2.png)
## 2. 安装 分析插件
> 点击配置-应用市场，搜索安装了java php js的相关插件，还安装了L10n，开始没有安装，pipeline后面编译maven示例的时候报错了，安装还是有必要的

![jenkins-sonar3](/assets/images/sonar/jenkins-sonar3.png)
## 2. 生成token
> token 字符串是用于 Jenkins 在执行流水线时候将待检测信息发送到 SonarQube的安全凭证。
> 点击右上角头像—我的账号—安全—生成令牌 生成验证的 Token。

![jenkins-sonar4](/assets/images/sonar/jenkins-sonar4.png)

# jenkins整合
## 1. 安装相关插件
>	
Maven Integration plugin
>
Pipeline Maven Integration Plugin
>
Pipeline Utility Steps
>
SonarQube Scanner for Jenkins
>
打开 系统管理—插件管理—可选插件 输入 相关插件名称 进行插件筛选，直接安装就Ok了。
## 2. 连接sonarqube
> 点击凭据-系统-全局凭证-添加凭据-Secret text，复制sonarqube token到Secret ID,命名为sonar了 我就.
![jenkins-sonar5](/assets/images/sonar/jenkins-sonar5.png)
> 系统管理-系统配置-SonarQube servers，添加sonarqube service配置。name 随便命名了一个jenkins,server url，由于我的jenkins和sonarqube 在一个namespace 我直接用了service 那么 通信，server authentication 添加了上一步创建的 sonar的secret text。
![jenkins-sonar6](/assets/images/sonar/jenkins-sonar6.png)
## 3. 配置 SonarQube Scanner 插件
>打开 系统管理—全局工具配置—SonarQube Scanner 输入 Name，选择最新版本点击自动安装即可.
![jenkins-sonar7](/assets/images/sonar/jenkins-sonar7.png)
## 4. 配置 maven插件
打开 系统管理—全局工具配置—Maven 输入 Name，选择最新版本点击自动安装即可.（我的安装的时候一直下不下来包，就直接下载了一个最新版的包copy到了容器中的路径中去.）
![jenkins-sonar8](/assets/images/sonar/jenkins-sonar8.png)
# 创建pipeline流水线测试项目
## 1. 创建流水线任务
![jenkins-sonar9](/assets/images/sonar/jenkins-sonar9.png)
### 2. 参数化构建流程-文本参数
 ```bash
名称： sonar_project_properties
默认值：
sonar.sources=src
sonar.language=java
sonar.sourceEncoding=UTF-8
sonar.java.binaries=target/classes
sonar.java.source=1.8
sonar.java.target=1.8
 ```
![jenkins-sonar10](/assets/images/sonar/jenkins-sonar10.png)
### 3. 创建pipeline 脚本 这里都是直接copy过来的
 ```bash
**// 设置超时时间为10分钟，如果未成功则结束任务
timeout(time: 600, unit: 'SECONDS') {
    node () {
        stage('Git 拉取阶段'){
            // Git 拉取代码
            git branch: "master" ,changelog: true , url: "https://github.com/a324670547/springboot-helloworld"
        }
        stage('Maven 编译阶段') {
            // 设置 Maven 工具,引用先前全局工具配置中设置工具的名称
            def m3 = tool name: 'maven'
            // 执行 Maven 命令
            sh "${m3}/bin/mvn -B -e clean install -Dmaven.test.skip=true"
        }
        stage('SonarQube 扫描阶段'){
            // 读取maven变量
            pom = readMavenPom file: "./pom.xml"
            // 创建SonarQube配置文件
            writeFile file: 'sonar-project.properties', 
                      text: """sonar.projectKey=${pom.artifactId}:${pom.version}\n"""+
                            """sonar.projectName=${pom.artifactId}\n"""+
                            """sonar.projectVersion=${pom.version}\n"""+
                            """${sonar_project_properties}"""
            // 设置 SonarQube 代码扫描工具,引用先前全局工具配置中设置工具的名称
            def sonarqubeScanner = tool name: 'sonar-scanner'
            // 设置 SonarQube 环境,其中参数设置为之前系统设置中SonarQuke服务器配置的 Name
            withSonarQubeEnv('jenkins') {
                // 执行代码扫描
                sh "${sonarqubeScanner}/bin/sonar-scanner"
            }
        }
    }
}**
  ```
![jenkins-sonar11](/assets/images/sonar/jenkins-sonar11.png)
### 4. 执行jenkins任务构建
>点击 Build with Parameters 执行 Jenkins 任务,由于插件安装不完整，sonarqube 少安装了L10n插件，开始失败率 好多次。等待确认成功
![jenkins-sonar12](/assets/images/sonar/jenkins-sonar12.png)
# 登陆sonarqube查看扫描结果.

![jenkins-sonar13](/assets/images/sonar/jenkins-sonar13.png)

> 总结：流程算是草草完成，还有很多不明白的地方，因为工作环境都是php，也没有成熟的发布流程，对java的maven构建还是很陌生。而且sonarqube的配置还是十分不熟悉。后续先搞明白下sonarqube的各种配置设置参数，系统的看下maven gradle这些主流的java构建工具。