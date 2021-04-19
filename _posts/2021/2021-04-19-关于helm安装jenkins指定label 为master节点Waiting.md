---
layout: post
title: 关于helm安装jenkins指定label 为master节点Waiting.
date: 2021-04-19 04:00:00
category: Jenkins
tags: Helm kubernetes
author: duiniwukenaihe
---
* content
{:toc}


helm 安装jenkins，做个测试 指定pipeline agent  label为master节点：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618824140147-439080a2-0b6e-4226-845d-214db00cdfe5.png#clientId=ubed6188c-8af5-4&from=paste&height=281&id=u0e2a5d51&margin=%5Bobject%20Object%5D&name=image.png&originHeight=281&originWidth=1200&originalType=binary&size=24047&status=done&style=none&taskId=ucfe24a5b-004f-4e2a-8a30-8a53a4b6dd1&width=1200)
```
pipeline{
    //指定运行此流水线的节点
    agent { label "master" }
    
    //管道运行选项
    options {
        skipStagesAfterUnstable()
    }
    //流水线的阶段
    stages{
        //阶段1 获取代码
        stage("CheckOut"){
            steps{
                script{
                    println("获取代码")
                    echo '你好张鹏'
                    sh "echo ${env.JOB_NAME}"
                }
            }
        }
        stage("Build"){
            steps{
                script{
                    println("运行构建")
                }
            }
        }
    }
    post {
        always{
            script{
                println("流水线结束后，经常做的事情")
            }
        }
        
        success{
            script{
                println("流水线成功后，要做的事情")
            }
        
        }
        failure{
            script{
                println("流水线失败后，要做的事情")
            }
        }
        
        aborted{
            script{
                println("流水线取消后，要做的事情")
            }
        
        }
    }
}

```
执行job时候等了好久，没有完成job。很诧异...就是几个简单的打印怎么会这样呢？当指定agent any时候很快就执行完成了。



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618824252213-9238b404-192b-40a2-9d5b-42f3a9055b15.png#clientId=ubed6188c-8af5-4&from=paste&height=340&id=uaacdeb75&margin=%5Bobject%20Object%5D&name=image.png&originHeight=340&originWidth=634&originalType=binary&size=30885&status=done&style=none&taskId=ud61f79d5-8cc8-45fc-b8a8-5e25254aa9a&width=634)


查看 Console output:  Waiting for next available executor on ‘[Jenkins] ’




![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618824382709-a5f8e18a-ee76-490d-bb3f-b4e25c44f688.png#clientId=ubed6188c-8af5-4&from=paste&height=694&id=uda3c204e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=694&originWidth=1505&originalType=binary&size=78566&status=done&style=none&taskId=u850c6462-d15f-4c7f-9d41-d5d671c299f&width=1505)




打开系统管理-节点管理-Master节点-配置从节点：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618824392526-a00f1f36-0ffa-42cb-9c4e-b590a3ebdfe3.png#clientId=ubed6188c-8af5-4&from=paste&height=709&id=u0319b30b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=709&originWidth=1695&originalType=binary&size=66196&status=done&style=none&taskId=u586d97fe-75b9-4cd8-9bbd-457d25b9a41&width=1695)
**嗯 默认的master的执行器数量是0，修改为5（这个真的是自己随意设置的，仅用于测试），OK保存。然后发现pipeline任务可以正常跑完了......**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618824459805-3b66de70-ab0f-4b16-ac40-3f75b162b1ab.png#clientId=ubed6188c-8af5-4&from=paste&height=749&id=ua1d0ea7a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=749&originWidth=1336&originalType=binary&size=95993&status=done&style=none&taskId=u11c97875-3d1f-4570-ba7b-6168dea86a9&width=1336)
遇事不慌，查看日志，根据日志找对应配置......
