---
layout: post
title: 2021-05-11-Jenkins Pipeline演进 
date: 2021-05-11 6:00:00
category: pipeline
tags: jenkins pipeline
author: duiniwukenaihe
---
* content
{:toc}

# 背景：
生产环境都部署在kubernetes集群上，使用jenkins打包镜像并部署在kubernetes集群中。关于jenkins的安装参照：[https://duiniwukenaihe.github.io/2019/11/19/k8s-install-jenkins/](https://duiniwukenaihe.github.io/2019/11/19/k8s-install-jenkins/)。当然了也有helm的安装方式[https://duiniwukenaihe.github.io/2021/03/31/Kubernetes-1.20.5-helm-%E5%AE%89%E8%A3%85jenkins/](https://duiniwukenaihe.github.io/2021/03/31/Kubernetes-1.20.5-helm-%E5%AE%89%E8%A3%85jenkins/)。
代码仓库是搭建的gitlab仓库。仓库中每个子项目多包含Dockerfile文件。镜像的tag采用了时间戳的命名方式：
```
return new Date().format('yyyyMMddHHmm')
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620696933020-12edced2-6467-4043-92f0-209f2312074a.png#clientId=ued98a1d3-a285-4&from=paste&height=653&id=u954f59e0&margin=%5Bobject%20Object%5D&name=image.png&originHeight=653&originWidth=1348&originalType=binary&size=52693&status=done&style=none&taskId=u612da52d-99c2-4a45-a64a-c9972799fb2&width=1348)
仓库的branch也可以定义一下：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620696979346-c237abaa-9aa4-40fb-ba79-6c1de9754bd7.png#clientId=ued98a1d3-a285-4&from=paste&height=619&id=u678251de&margin=%5Bobject%20Object%5D&name=image.png&originHeight=619&originWidth=1056&originalType=binary&size=42725&status=done&style=none&taskId=u086de5bb-bbab-41ca-bd22-a9a9f63b3af&width=1056)
当然了 这里有两个插件要安装的：
### 1. dynamic parameter
这是一个非常古老的插件了。但是还比较好用。下载地址：[http://mirror.xmission.com/jenkins/plugins/dynamicparameter/](http://mirror.xmission.com/jenkins/plugins/dynamicparameter/) 。安装的方式是上传：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620697156781-35e1067a-8c6f-488d-bcff-06c9b684e1f4.png#clientId=ued98a1d3-a285-4&from=paste&height=854&id=u79e35106&margin=%5Bobject%20Object%5D&name=image.png&originHeight=854&originWidth=1696&originalType=binary&size=55346&status=done&style=none&taskId=u810fed66-de43-4d67-bea9-1920627a5d0&width=1696)
### 2. Git Parameter（我是早安装了在初始化的时候）
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620697470762-f5122344-0e3f-4b81-ab16-12115791c0c8.png#clientId=ued98a1d3-a285-4&from=paste&height=656&id=u193609ae&margin=%5Bobject%20Object%5D&name=image.png&originHeight=656&originWidth=1511&originalType=binary&size=99202&status=done&style=none&taskId=u1e0e20a8-87d1-4e91-b252-e656efc2424&width=1511)
# 演进过程：
## 1. 看一下早些时候写的pipeline：
仓库是自己搞的 直接先xxxx了。偷懒写的明文用户名密码，docker image仓库直接使用的腾讯云的镜像仓库个人版。
```
echo env.data 
pipeline {
    agent any
    parameters {
        gitParameter branchFilter: 'origin/(.*)', defaultValue: 'develop', name: 'BRANCH', type: 'PT_BRANCH'
    }
    stages {
        stage('Checkout') {
            agent { node { label 'xxxx-jnlp'} }            
            steps {
                sh '''
                git clone --depth=1  -b cache http://xxx:xxx@github.com/xxx/xxx.git'''
    
            }
        } 
        stage('docker build dataloader') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/dataloader
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
					             docker build --build-arg NODE_ENV=game-ucenter -t ccr.ccs.tencentyun.com/xxxx-master/dataloader-game-ucenter:$data .					   
					             docker push ccr.ccs.tencentyun.com/xxxx-master/dataloader-game-ucenter:$data'''
            }
            }
        stage('docker build datawriter-old') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/datawriter-old
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
					             docker build --build-arg NODE_ENV=game-ucenter -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-game-ucenter:$data .					   
					             docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-game-ucenter'''
            }
            }		
        stage('docker build datawriter') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/datawriter
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build --build-arg NODE_ENV=xxxx-maker -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-maker:$data .
					             docker build --build-arg NODE_ENV=xxxx-comment -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-comment:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-maker:$data
					             docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-comment:$data'''
            }
            }			
        stage('docker build game-ucenter') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/game-ucenter
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/game-ucenter:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/game-ucenter:$data'''
            }
            }         
        stage('docker build xxxx-comment') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/xxxx-comment
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-comment:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-comment:$data'''
            }
            }
        stage('docker build xxxx-maker') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/xxxx-maker
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-maker:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-maker:$data'''
            }
            }
        stage('docker build MessageConsumer') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/messageconsumer
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build --build-arg NODE_ENV=xxxx-maker -t ccr.ccs.tencentyun.com/xxxx-master/messageconsumer:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/messageconsumer:$data'''
            }
            }
        stage('docker build xxxxtimer ') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/xxxxtimer 
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxxtimer:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/xxxxtimer:$data'''
            }
            }            
          stage('docker build mail-koa ') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/mail-koa 
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/mail-koa:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/mail-koa:$data'''
            }
            }  
        stage('docker build xxxx-diversion ') {
            agent { node { label 'xxxx-jnlp'} }
            steps {
                sh ''' cd /home/jenkins/agent/workspace/develop-xxxxme/xxxxme/diversion
                       docker login --username=xxxxxx ccr.ccs.tencentyun.com -p xxxxx
                       docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-diversion:$data .				   
                       docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-diversion:$data'''
            }
            }             
        stage('项目部署') {
            agent { node { label 'build01'} }
            steps {
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-comment.tpl > /home/jenkins/workspace/yaml/develop/datawriter-comment.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-maker.tpl > /home/jenkins/workspace/yaml/develop/datawriter-maker.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/game-ucenter.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-comment.tpl > /home/jenkins/workspace/yaml/develop/xxxx-comment.yaml"		
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-maker.tpl > /home/jenkins/workspace/yaml/develop/xxxx-maker.yaml"	
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/messageconsumer.tpl > /home/jenkins/workspace/yaml/develop/messageconsumer.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxxtimer.tpl > /home/jenkins/workspace/yaml/develop/xxxxtimer.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/mail-koa.tpl > /home/jenkins/workspace/yaml/develop/mail-koa.yaml"
                sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-diversion.tpl > /home/jenkins/workspace/yaml/develop/xxxx-diversion.yaml"    
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-comment.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-maker.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/game-ucenter.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-comment.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-maker.yaml --namespace=develop"	
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/messageconsumer.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxxtimer.yaml --namespace=develop"
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/mail-koa.yaml --namespace=develop"  
                sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-diversion.yaml --namespace=develop"
            }
            }
}
}			
```
这样的一个pipeline 算是解决了我初期的问题。能直接一步发版了。但是在后面的使用中出现了下面的一些问题：
​

### 1. git 分支切换。虽然上面写了Git Parameter插件，但是其实我并没有使用。


### 2. 如何快速切换git地址。我可能有两个或者三个的git仓库。


### 3. gitlab  harbor仓库秘钥明文，这样很是不安全。


### 4. 我的stage 有十多个。其实我这次只更新了一个项目，我只想更新单个的stage，我想有选择的标签或者按钮，去选择我要更新的子项目。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620699282948-23c746d7-d48e-4730-8649-d78d88784808.png#clientId=ued98a1d3-a285-4&from=paste&height=734&id=u90d03674&margin=%5Bobject%20Object%5D&name=image.png&originHeight=734&originWidth=1920&originalType=binary&size=141449&status=done&style=none&taskId=u8dcf61aa-1f70-48aa-838c-ece261bf403&width=1920)




### 5. 抛弃下早期的构建，设置保留的天数和次数。（任务数太多了数量也，且无用）


## 2. 进化过程：


### 1. git 分支的切换问题
#### 1. 针对git 分支切换：我并没有去使用Git Parameter的插件。使用了字符参数的方式：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620699592384-31f8e07b-2e59-499a-956b-e6026901fba0.png#clientId=ued98a1d3-a285-4&from=paste&height=446&id=u13c53afa&margin=%5Bobject%20Object%5D&name=image.png&originHeight=446&originWidth=937&originalType=binary&size=22982&status=done&style=none&taskId=u7c9ed4d4-94d1-4819-a072-0f18b43524a&width=937)
使用Git Parameter插件的话，新建的分支应该是要拉取一次后才能有这个参数的。对于我来说不利于迭代，因为程序也经常建立一些无规则的分支。我就设置了默认的cache分支，然后切换分支的话让他自己定义就好了。
#### 2. 切换选择git仓库，使用了选项参数。这个地方是固定的对于程序来说就只能固定的这两个仓库的：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620699877310-caef1d05-a007-4410-911c-3ab56ae0b2db.png#clientId=ued98a1d3-a285-4&from=paste&height=554&id=u1d5654be&margin=%5Bobject%20Object%5D&name=image.png&originHeight=554&originWidth=1120&originalType=binary&size=39805&status=done&style=none&taskId=u3df33c35-61fc-4bc2-9214-0215f1ff2f0&width=1120)
还整了一个选项参数：srcType .copy泽阳老师课程来的。虽然仓库默认都是gitlab了。但是凑个数添加一下svn。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620699952656-90ab5e93-b6c4-46e8-89a9-44998179130d.png#clientId=ued98a1d3-a285-4&from=paste&height=577&id=ubd05d41d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=577&originWidth=1126&originalType=binary&size=34813&status=done&style=none&taskId=uce2eed73-77a9-47c5-9ae5-5ed95e08520&width=1126)
2. 关于凭据秘钥
将仓库 还有gitlab秘钥都在jenkins中新建为秘钥：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620700171784-7b1f3848-91ec-4d41-87f3-91190e4bdaad.png#clientId=ued98a1d3-a285-4&from=paste&height=660&id=ud666a1b9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=660&originWidth=1482&originalType=binary&size=100096&status=done&style=none&taskId=u43661958-6eee-47ad-995a-e9a26d012b5&width=1482)先把拉取代码的stage上一下：
```
 		stage("GetCode"){
		    agent { label  "build" }
			steps{
				script{
					println("下载代码 --> 分支： ${env.branchName}")
 					checkout([$class: 'GitSCM', branches: [[name: "${env.branchName}"]],
 							 extensions: [], 
 							 userRemoteConfigs: [[credentialsId: 'XXXX', 
 							 						url: "${env.gitHttpURL}"]]])
				}
			}
         } 
```
### 3. stage 的单选多选问题
至于stage的发布选取[https://www.cnblogs.com/jwentest/p/7113399.html](https://www.cnblogs.com/jwentest/p/7113399.html)看到了这样的复选框的方式看了下
，觉得貌似可以用。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620721679889-8a935693-53c5-40cc-8f99-a63c129710c6.png#clientId=u81c08dd0-ab1f-4&from=paste&height=342&id=ucc8414a1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=342&originWidth=1101&originalType=binary&size=20066&status=done&style=none&taskId=ue2e0a3e4-71c1-4c6b-9581-3740b0da81c&width=1101)
也做了这样的测试，但是到pipeline里面是不是要转成列表还要循环？这样的步骤个人不熟悉。放弃了。
最终用了另外一种方式：对于每一个子项目。我都添加了一个布尔值参数。然后只给常用的加上了 默认值。每个stage做一个when的判断：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620701039598-045c51d2-5520-4530-b7c2-37d5c660811d.png#clientId=ued98a1d3-a285-4&from=paste&height=726&id=u0fe3bd89&margin=%5Bobject%20Object%5D&name=image.png&originHeight=726&originWidth=1316&originalType=binary&size=47528&status=done&style=none&taskId=uf51e2796-dfe2-4265-9b9e-5f8504238b5&width=1316)
基本就是这样的，然后我的pipeline直接用上了 :
```
            when {
                environment name: 'XXXX', value: 'true'
            }
```
这样就能解决了选择发布发布的选择问题。然后就说dockerhub秘钥的明文问题了：
```
        stage('docker build XXX') {
            agent { label  "build" }
            when {
                environment name: 'xxxx', value: 'true'
            }
            steps {
                sh " cd XXX&&docker build -t ccr.ccs.tencentyun.com/xxxx/XXXX:$data ."
                withCredentials([usernamePassword(credentialsId: 'XXXXX', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxx-master/xxxx:$data"
        }       
            }
            }
```
然后就说我的发布流程根据tpl模板生成yaml 文件然后apply就做成了并行的stage:
```
parallel {
}
```
如下：

```
pipeline {
    
	agent { label  "build" }
	stages {
 		stage("GetCode"){
		    agent { label  "build" }
			steps{
				script{
					println("下载代码 --> 分支： ${env.branchName}")
 					checkout([$class: 'GitSCM', branches: [[name: "${env.branchName}"]],
 							 extensions: [], 
 							 userRemoteConfigs: [[credentialsId: 'xxxx-open-php', 
 							 						url: "${env.gitHttpURL}"]]])
				}
			}
         } 
        stage('docker build dataloader') {
            agent { label  "build" }
            when {
                environment name: 'dataloader', value: 'true'
            }
            steps {
                sh ''' cd dataloader
                       docker login --username=xxxx ccr.ccs.tencentyun.com -p Aa123456.
                       docker build --build-arg NODE_ENV=xxxx-maker -t ccr.ccs.tencentyun.com/xxxx-master/dataloader-maker:$data .
					             docker build --build-arg NODE_ENV=xxxx-comment -t ccr.ccs.tencentyun.com/xxxx-master/dataloader-comment:$data .
                       docker build --build-arg NODE_ENV=game-ucenter -t ccr.ccs.tencentyun.com/xxxx-master/dataloader-game-ucenter:$data .'''
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/dataloader-maker:$data"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/dataloader-comment:$data"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/dataloader-game-ucenter:$data"


        }             			
            }
            }
        stage('docker build datawriter-old') {
            agent { label  "build" }
            when {
                environment name: 'datawriter-old', value: 'true'
            }
            steps {
                sh " cd datawriter-old&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-old:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-old:$data"
        }             
            }
            }		
        stage('docker build datawriter') {
            agent { label  "build" }
            when {
                environment name: 'datawriter', value: 'true'
            }
            
            steps {
                sh ''' cd datawriter
                       docker login --username=xxxx ccr.ccs.tencentyun.com -p Aa123456.
                       docker build --build-arg NODE_ENV=xxxx-maker -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-maker:$data .
					             docker build --build-arg NODE_ENV=xxxx-comment -t ccr.ccs.tencentyun.com/xxxx-master/datawriter-comment:$data .'''
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-maker:$data"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/datawriter-comment:$data"


        }             
            }
            }			
        stage('docker build game-ucenter') {
            agent { label  "build" }
            when {
                environment name: 'game-ucenter', value: 'true'
            }
            steps {
                sh " cd game-ucenter&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/game-ucenter:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/game-ucenter:$data"
        }             
            }
            }         
        stage('docker build xxxx-comment') {
            agent { label  "build" }
            when {
                environment name: 'xxxx-comment', value: 'true'
            }
            steps {
                sh " cd xxxx-comment&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-comment:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-comment:$data"
        }             
            }
            }
        stage('docker build xxxx-maker') {
            agent { label  "build" }
            when {
                environment name: 'xxxx-maker', value: 'true'
            }
            steps {
                sh " cd xxxx-maker&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-maker:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-maker:$data"
        }       
            }
            }
        stage('docker build MessageConsumer') {
            agent { label  "build" }
            when {
                environment name: 'MessageConsumer', value: 'true'
            }
            steps {
                sh " cd messageconsumer&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/messageconsumer:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/messageconsumer:$data"
            }
            }
        }
        stage('docker build xxxxtimer ') {
            agent { label  "build" }
            when {
                environment name: 'xxxxtimer', value: 'true'
            }
            steps {
                 sh " cd xxxxtimer&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxxtimer:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/xxxxtimer:$data"               
            }
            } 
        }           
          stage('docker build mail-koa ') {
            agent { label  "build" }
            when {
                environment name: 'mail-koa', value: 'true'
            }
            steps {
                sh " cd mail-koa&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/mail-koa:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/mail-koa:$data"
            }            
            }
            }  
        stage('docker build xxxx-diversion ') {
            agent { label  "build" }
            when {
                environment name: 'diversion', value: 'true'
            }
            steps {
                sh " cd diversion&&docker build -t ccr.ccs.tencentyun.com/xxxx-master/xxxx-diversion:$data ."
                withCredentials([usernamePassword(credentialsId: 'xxxx', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                    sh "docker login -u ${dockerUser} -p ${dockerPassword} ccr.ccs.tencentyun.com"
                    sh "docker push ccr.ccs.tencentyun.com/xxxx-master/xxxx-diversion:$data"
            }             
            }
            }
        stage('develop') {
            parallel {
                stage("develop dataloader") {
                    when {
                        environment name: 'dataloader', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/dataloader-comment.tpl > /home/jenkins/workspace/yaml/develop/dataloader-comment.yaml"
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.yaml"
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/dataloader-maker.tpl > /home/jenkins/workspace/yaml/develop/dataloader-maker.yaml"
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/dataloader-game-ucenter.yaml --namespace=develop"

}
      }
                stage("develop datawriter-old") {
                    when {
                        environment name: 'datawriter-old', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.yaml"
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-game-ucenter.yaml --namespace=develop"
}
      } 
                stage("develop datawriter") {
                    when {
                        environment name: 'datawriter', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-maker.tpl > /home/jenkins/workspace/yaml/develop/datawriter-maker.yaml"
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/datawriter-comment.tpl > /home/jenkins/workspace/yaml/develop/datawriter-comment.yaml"
			                  sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-comment.yaml --namespace=develop"
			                  sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/datawriter-maker.yaml --namespace=develop"
}
      }
                stage("develop game-ucenter") {
                    when {
                        environment name: 'game-ucenter', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/game-ucenter.tpl > /home/jenkins/workspace/yaml/develop/game-ucenter.yaml"
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/game-ucenter.yaml --namespace=develop"
}
      }
                stage("develop xxxx-comment") {
                    when {
                        environment name: 'xxxx-comment', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-comment.tpl > /home/jenkins/workspace/yaml/develop/xxxx-comment.yaml"	
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-comment.yaml --namespace=develop"

}
      }				
                stage("develop xxxx-maker") {
                    when {
                        environment name: 'xxxx-maker', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-maker.tpl > /home/jenkins/workspace/yaml/develop/xxxx-maker.yaml"		
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-maker.yaml --namespace=develop"

}
      }	
                stage("develop MessageConsumer") {
                    when {
                        environment name: 'messageconsumer', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/messageconsumer.tpl > /home/jenkins/workspace/yaml/develop/messageconsumer.yaml"		
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/messageconsumer.yaml --namespace=develop"

}
      }					
                stage("develop xxxxtimer") {
                    when {
                        environment name: 'xxxxtimer', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxxtimer.tpl > /home/jenkins/workspace/yaml/develop/xxxxtimer.yaml"	
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxxtimer.yaml --namespace=develop"

}
      }					
                stage("develop mail-koa") {
                    when {
                        environment name: 'mail-koa', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/mail-koa.tpl > /home/jenkins/workspace/yaml/develop/mail-koa.yaml"
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/mail-koa.yaml --namespace=develop" 

}
      }		
                stage("develop xxxx-diversion") {
                    when {
                        environment name: 'xxxx-diversion', value: 'true'
                    }
                 
                    steps {
                        sh "sed -e 's/{data}/$data/g' /home/jenkins/workspace/yaml/develop/xxxx-diversion.tpl > /home/jenkins/workspace/yaml/develop/xxxx-diversion.yaml"
                        sh "sudo kubectl apply -f /home/jenkins/workspace/yaml/develop/xxxx-diversion.yaml --namespace=develop"

}
      }      			
				

            }
            }             
	}
}
```
### 4. 关于抛弃旧的构建：
直接偷懒在web 上设置了：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620722003507-1c0b50e7-387b-4d57-b992-a1943e9dc020.png#clientId=u81c08dd0-ab1f-4&from=paste&height=710&id=ucfcf08f1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=710&originWidth=1505&originalType=binary&size=60095&status=done&style=none&taskId=u5a0a3b46-2a07-480b-8cd4-6325e437d8c&width=1505)
当然了也可以在pipeline中设置：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620722159655-c642e3c8-0212-4436-8446-087eb1c3de4d.png#clientId=u81c08dd0-ab1f-4&from=paste&height=672&id=u534f3300&margin=%5Bobject%20Object%5D&name=image.png&originHeight=672&originWidth=1207&originalType=binary&size=98675&status=done&style=none&taskId=u80eb9cb0-b095-4963-85df-0c89bfbad3f&width=1207)
选择天数和最大个数：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620722186770-8891ecce-0828-49f1-865f-197ce2a6d70d.png#clientId=u81c08dd0-ab1f-4&from=paste&height=753&id=u76af37cb&margin=%5Bobject%20Object%5D&name=image.png&originHeight=753&originWidth=1727&originalType=binary&size=102405&status=done&style=none&taskId=u22fa8fce-3821-449c-99fe-3a42e90cfc1&width=1727)
将生成的option放入pipeline脚本即可。做完了测试了一下：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620722358634-b0939607-c94e-4bab-a9cf-d4dc8193ce2a.png#clientId=u81c08dd0-ab1f-4&from=paste&height=707&id=u1ba8b166&margin=%5Bobject%20Object%5D&name=image.png&originHeight=707&originWidth=1761&originalType=binary&size=93353&status=done&style=none&taskId=u2b5b0ae9-ef8f-46cf-9e23-a830d6b279d&width=1761)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620722311349-6f62b92f-8b41-494a-b1c8-f91271b7e257.png#clientId=u81c08dd0-ab1f-4&from=paste&height=590&id=uab99fafe&margin=%5Bobject%20Object%5D&name=image.png&originHeight=590&originWidth=1899&originalType=binary&size=141802&status=done&style=none&taskId=u6b11f37e-1fab-45ba-860d-39f292391bf&width=1899)
算是基本满足自己的需求了，这算是学了泽阳大佬的jenkins课程后改的自己过去写的第一个pipeline。包括很多步骤都没有加。比如代码扫描，程序测试流程。还有构建后邮件或者钉钉通知。先把这流水线改的顺眼一些吧....另外这周when的判断还是有点抵触，后面看看能不能有更好的方法去简练一些pipeline呢。当前就是看着顺眼能跑。


