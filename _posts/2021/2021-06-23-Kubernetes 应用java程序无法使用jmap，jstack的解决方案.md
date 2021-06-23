---
layout: post
title: 2021-06-23-Kubernetes 应用java程序无法使用jmap，jstack的解决方案
date: 2021-06-23 2:00:00
category: kubernetes
tags: java jstack jmap
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

基础环境[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://cloud.tencent.com/developer/article/1806089?from=10680)，线上主要跑的php nodejs java的环境。

java的pod昨天频繁出现了cpu 90%的占用率告警：

![image.png](/assets/images/2021/06-23/mopq949qvy.png)

虽然cpu是可压缩资源（compressible resources ）,应用只会饥饿，不会像是内存爆了一样OOM.但是也需要进行一下性能分析，看一眼是代码逻辑有问题，还是资源分配的大小不合理。

就想传统的方式进入容器查看pid，运行jstack命令进行分析了：

![](/assets/images/2021/06-23/83ame0h5d7.png)

```
kubectl exec -it xxx-xxxx-8556c7f98b-9nh28 sh -n official
/ # ps
PID   USER     TIME  COMMAND
    1 root      2h38 java -Djava.security.egd=file:/dev/./urandom -jar /xxx-1.0-SNAPSHOT.jar
 4178 root      0:00 sh
 4193 root      0:00 ps
/ # jstack 1
1: Unable to get pid of LinuxThreads manager thread
```

what  jstack命令无法分析应用......

# 解决过程：

百度搜索了一下，参见：

[https://blog.csdn.net/qq\_16887777/article/details/107417059](https://blog.csdn.net/qq_16887777/article/details/107417059)

## 1. 关于pid1

发现服务的pid=1，网上查询得知pid1-5为Linux的特殊进程。

pid=1 ：init进程，系统启动的第一个用户级进程，是所有其它进程的父进程，引导用户空间服务。

pid=2 ：kthreadd：用于内核线程管理。

pid=3 ：migration，用于进程在不同的CPU间迁移。

pid=4 ：ksoftirqd，内核里的软中断守护线程，用于在系统空闲时定时处理软中断事务。

pid=5 ：watchdog，此进程是看门狗进程，用于监听内核异常。当系统出现宕机，可以利用watchdog进程将宕机时的一些堆栈信息写入指定文件，用于事后分析宕机的原因。

根据排除法最简单的方式就是让java启动的进程pid不是1-5就可以了？嗯启动命令不是第一个。

## 2. 修改Dcokerfile java启动方式

上下我的Dockerfile

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ADD target/diversion-0.0.1-SNAPSHOT.jar diversion-0.0.1-SNAPSHOT.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/xxxx-0.0.1-SNAPSHOT.jar"]
```

嗯个人认为可以将 java 启动文件写入脚本？然后ENTRYPOINT sh脚本？偶然看到一个tini的方法：[docker运行java程序 使用jmap，jstack命令 tini运行的程序获取进程](https://blog.csdn.net/dengjiaqun/article/details/106836546?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_v2~rank_aggregation-1-106836546.pc_agg_rank_aggregation&utm_term=linux+tini+%25E5%2591%25BD%25E4%25BB%25A4&spm=1000.2123.3001.4430).修改Dockerfile如下：

```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ENV TZ=Asia/Shanghai
RUN apk add --no-cache tini
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ADD target/diversion-0.0.1-SNAPSHOT.jar diversion-0.0.1-SNAPSHOT.jar
ENTRYPOINT ["tini","java","-Djava.security.egd=file:/dev/./urandom","-jar","/xxxx-0.0.1-SNAPSHOT.jar"]
```

## 3. 构建并上传镜像到镜像仓库

```
docker build -t ccr.ccs.tencentyun.com/xxxx/xxxx:xxxx
docker push ccr.ccs.tencentyun.com/xxxx/xxxx:xxxx
```

## 4. 部署应用并测试

```
cat > test.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: test
          image: ccr.ccs.tencentyun.com/xxxx/xxxx:xxxx
          env:
          - name: SPRING_PROFILES_ACTIVE
            value: "official"
          ports:
            - containerPort: test
          resources:
            requests:
              memory: "256M"
              cpu: "250m"
            limits:
              memory: "1024M"
              cpu: "500m" 
      imagePullSecrets:                                              
        - name: tencent
---

apiVersion: v1
kind: Service
metadata:
  name: test
  labels:
    app: test
spec:
  ports:
  - port: 8081
    protocol: TCP
    targetPort: 8081
  selector:
    app: test
EOF
```

```
kubectl apply -f test.yaml -n test
kubectl exec -it xxxxxxx sh -n test
top
```

![image.png](/assets/images/2021/06-23/91gtm6rvzw.png)

可以看到进程号是1的进程是tini的 有额外一个单独的进程号为7的java 进程，运行jstack进行测试：

jstack 7

![image.png](/assets/images/2021/06-23/3a7i9ktrjf.png)

嗯能运行jstack就算是实现了自己需要的了。其他的先忽略。

# 总结：

## 1. 关于linux的pid

## 2. tini的命令，可参照[https://zhuanlan.zhihu.com/p/59796137](https://zhuanlan.zhihu.com/p/59796137)

## 3. docker的启动方式与进程隔离实现方式