---
layout: post
title: Kubernetes 1.20.5搭建sentinel
date: 2021-04-14 8:00:00
category: kubernetes1.20 
tags:  kubernetes sentinel springboot
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
后端程序团队准备一波流springcloud架构,配置中心用了阿里开源的nacos。不出意外的要高一个sentinel做测试了.....。做一个demo开始搞一下吧。
kubernetes上面搭建sentinel的案例较少。看下眼还是springcloud全家桶的多点。阿里开源的这一套还是少点。百度或者Google搜索 sentinel基本出来的都是redis哨兵模式...有点忧伤。
注：搭建方式可以参照：[https://blog.csdn.net/fenglailea/article/details/92436337?utm_term=k8s%E9%83%A8%E7%BD%B2Sentinel&utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduweb~default-0-92436337&spm=3001.4430](https://blog.csdn.net/fenglailea/article/details/92436337?utm_term=k8s%E9%83%A8%E7%BD%B2Sentinel&utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~all~sobaiduweb~default-0-92436337&spm=3001.4430)。
# 一. 搭建sentinel-dashboard:
## 1.自定义创建sentinel-dashboard image镜像
嗯 很理所当然了不喜欢用docker镜像这名词了。还是用image吧。上面引用的博客中1.6.1的版本吧 ？搭建跑了下犯了强迫症，最新的版本是1.8.1根据foxiswho大佬的配置文件进行修改下镜像。
vim Dockerfile
```
FROM openjdk:11.0.3-jdk-stretch

MAINTAINER foxiswho@gmail.com

ARG version
ARG port

# sentinel version
ENV SENTINEL_VERSION ${version:-1.8.1}
#PORT
ENV PORT ${port:-8858}
ENV JAVA_OPT=""
#
ENV PROJECT_NAME sentinel-dashboard
ENV SERVER_HOST localhost
ENV SERVER_PORT 8858
ENV USERNAME sentinel
ENV PASSWORD sentinel


# sentinel home
ENV SENTINEL_HOME  /opt/
ENV SENTINEL_LOGS  /opt/logs

#tme zone
RUN rm -rf /etc/localtime \
&& ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# create logs
RUN mkdir -p ${SENTINEL_LOGS}

# get the version
#RUN cd /  \
# && wget https://github.com/alibaba/Sentinel/releases/download/${SENTINEL_VERSION}/sentinel-dashboard-${SENTINEL_VERSION}.jar -O sentinel-dashboard.jar \
# && mv sentinel-dashboard.jar ${SENTINEL_HOME} \
# && chmod -R +x ${SENTINEL_HOME}/*jar
# test file
COPY sentinel-dashboard.jar ${SENTINEL_HOME}

# add scripts
COPY scripts/* /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh \
&& ln -s /usr/local/bin/docker-entrypoint.sh /opt/docker-entrypoint.sh

#
RUN chmod -R +x ${SENTINEL_HOME}/*jar

VOLUME ${SENTINEL_LOGS}

WORKDIR  ${SENTINEL_HOME}

EXPOSE ${PORT} 8719


CMD java ${JAVA_OPT} -jar sentinel-dashboard.jar

ENTRYPOINT ["docker-entrypoint.sh"]

```
注： 大佬的Dockerfile中 暴露的端口是8200，由于看sentinel对外暴露端口都是8518我就把dockerfile修改了。然后把[https://github.com/alibaba/Sentinel/releases](https://github.com/alibaba/Sentinel/releases)下载了1.8.1版本的jar包改名为sentinel-dashboard.jar 放在当前目录
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618384047494-b01c056f-05a7-4a70-91d3-2b23d46ee36c.png#align=left&display=inline&height=446&margin=%5Bobject%20Object%5D&name=image.png&originHeight=891&originWidth=1463&size=133906&status=done&style=none&width=731.5)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618384219265-c0ecd7ed-bff1-4962-a935-175a88890f8f.png#align=left&display=inline&height=87&margin=%5Bobject%20Object%5D&name=image.png&originHeight=173&originWidth=1626&size=24382&status=done&style=none&width=813)
dc目录可以忽略，原项目copy自[https://github.com/foxiswho/docker-sentinel](https://github.com/foxiswho/docker-sentinel)
cat scripts/docker-entrypoint.sh
```
#!/bin/bash

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
# Get the max heap used by a jvm, which used all the ram available to the container.
if [ -z "$MAX_POSSIBLE_HEAP" ]
then
	MAX_POSSIBLE_RAM_STR=$(java -XX:+UnlockExperimentalVMOptions -XX:MaxRAMFraction=1 -XshowSettings:vm -version 2>&1 | awk '/Max\. Heap Size \(Estimated\): [0-9KMG]+/{ print $5}')
	MAX_POSSIBLE_RAM=$MAX_POSSIBLE_RAM_STR
	CAL_UNIT=${MAX_POSSIBLE_RAM_STR: -1}
	if [ "$CAL_UNIT" == "G" -o "$CAL_UNIT" == "g" ]; then
		MAX_POSSIBLE_RAM=$(echo ${MAX_POSSIBLE_RAM_STR:0:${#MAX_POSSIBLE_RAM_STR}-1} `expr 1 \* 1024 \* 1024 \* 1024` | awk '{printf "%d",$1*$2}')
	elif [ "$CAL_UNIT" == "M" -o "$CAL_UNIT" == "m" ]; then
		MAX_POSSIBLE_RAM=$(echo ${MAX_POSSIBLE_RAM_STR:0:${#MAX_POSSIBLE_RAM_STR}-1} `expr 1 \* 1024 \* 1024` | awk '{printf "%d",$1*$2}')
	elif [ "$CAL_UNIT" == "K" -o "$CAL_UNIT" == "k" ]; then
		MAX_POSSIBLE_RAM=$(echo ${MAX_POSSIBLE_RAM_STR:0:${#MAX_POSSIBLE_RAM_STR}-1} `expr 1 \* 1024` | awk '{printf "%d",$1*$2}')
	fi
	MAX_POSSIBLE_HEAP=$[MAX_POSSIBLE_RAM/4]
fi

# Dynamically calculate parameters, for reference.
Xms=$MAX_POSSIBLE_HEAP
Xmx=$MAX_POSSIBLE_HEAP
Xmn=$[MAX_POSSIBLE_HEAP/2]
# Set for `JAVA_OPT`.
JAVA_OPT="${JAVA_OPT} -server "
if [ x"${MAX_POSSIBLE_HEAP_AUTO}" = x"auto" ];then
    JAVA_OPT="${JAVA_OPT} -Xms${Xms} -Xmx${Xmx} -Xmn${Xmn}"
fi
#-XX:+UseCMSCompactAtFullCollection
#JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 "
#JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
#JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
#JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
#JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} -Dserver.port=${PORT} "
JAVA_OPT="${JAVA_OPT} -Dcsp.sentinel.log.dir=${SENTINEL_LOGS} "
JAVA_OPT="${JAVA_OPT} -Djava.security.egd=file:/dev/./urandom"
JAVA_OPT="${JAVA_OPT} -Dproject.name=${PROJECT_NAME} "
JAVA_OPT="${JAVA_OPT} -Dcsp.sentinel.app.type=1 "
JAVA_OPT="${JAVA_OPT} -Dsentinel.dashboard.auth.username=${USERNAME} "
JAVA_OPT="${JAVA_OPT} -Dsentinel.dashboard.auth.password=${PASSWORD} "
JAVA_OPT="${JAVA_OPT} -Dcsp.sentinel.dashboard.server=${SERVER_HOST:-localhost}:${SERVER_PORT:-8558} "
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -jar sentinel-dashboard.jar "
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"
echo "JAVA_OPT============"
echo "JAVA_OPT============"
echo "JAVA_OPT============"
echo $JAVA_OPT

$JAVA ${JAVA_OPT} $@

```
依然抄的大佬的启动文件。但是注意...大佬这里也写死了端口8200....记得修改


嗯 开始build镜像 
```
docker build -t ccr.ccs.tencentyun.com/xxxx/sentinel:1.8.1 .
docker push ccr.ccs.tencentyun.com/xxxx/sentinel:1.8.1
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618384309523-5e0e7ddb-6284-4ced-9c7f-7102e52fa319.png#align=left&display=inline&height=420&margin=%5Bobject%20Object%5D&name=image.png&originHeight=840&originWidth=1004&size=93480&status=done&style=none&width=502)
对了 我是不是可以用crictl命令操作一下？crictl ctr 不支持build........后续是不是可以考虑用
buildkit 构建镜像？
## 2. 在kubernetes集群中部署sentinel
在[Kubernetes 1.20.5搭建nacos](https://www.yuque.com/duiniwukenaihe/ehb02i/vsg9vd)中建立了nacos namespace.  sentinel也部署在sentinel命名空间了没有做太多的复杂配置。这边都是简单跑起来的demo，先跑通流程
### 1. 部署configmap
**cat config.yaml**
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sentinel-cm
data:
  sentinel.server.host: "sentinel"
  sentinel.server.port: "8858"
  sentinel.dashboard.auth.username: "sentinel111111"
  sentinel.dashboard.auth.password: "W3$ti$aifffdfGEqjf.xOkZ"
```
注：这里的sentinel.server.host  我这里直接写的是服务名，还没有出现什么异常启动了。正常是不是该输入一个fqdn?   sentinel.nacos.svc.cluster.local  这样的呢？（当然了 我的domain不是cluster.local）.
```
kubectl apply -f config.yaml -n nacos
```
### 2 部署 sentinel statefulset
**cat pod.yaml**
```
apiVersion: apps/v1
kind: StatefulSet
metadata:

  name: sentinel
  labels:
    app: sentinel
spec:
  serviceName: sentinel
  replicas: 1
  selector:
    matchLabels:
      app: sentinel
  template:
    metadata:
      labels:
        app: sentinel
    spec:
      containers:
        - name: sentinel
          image: ccr.ccs.tencentyun.com/XXXX/sentinel:1.8.1
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 450m
              memory: 1024Mi
            requests:
              cpu: 400m
              memory: 1024Mi
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: JAVA_OPT_EXT
              value: "-Dserver.servlet.session.timeout=7200 "
            - name: SERVER_HOST
              valueFrom:
                configMapKeyRef:
                  name: sentinel-cm
                  key: sentinel.server.host
            - name: SERVER_PORT
              valueFrom:
                configMapKeyRef:
                  name: sentinel-cm
                  key: sentinel.server.port
            - name: USERNAME
              valueFrom:
                  configMapKeyRef:
                    name: sentinel-cm
                    key: sentinel.dashboard.auth.username
            - name: PASSWORD
              valueFrom:
                  configMapKeyRef:
                    name: sentinel-cm
                    key: sentinel.dashboard.auth.password
          ports:
            - containerPort: 8858
            - containerPort: 8719
          volumeMounts:
            - name: vol-log
              mountPath: /opt/logs
      volumes:
        - name: vol-log
          hostPath:
            path: /www/k8s/foxdev/sentinel/logs
            type: Directory

```
```
kubectl  apply -f pod.yaml -n nacos
```
注意：偷懒了 volumes挂载不想搞了本来就是测试的 在三个work节点都搞了/www/k8s/foxdev/sentinel/logs目录.直接复制foxiswho的配置了基本。
### 3. 部署service服务
**cat svc.yaml**
```
apiVersion: v1
kind: Service
metadata:

  name: sentinel
  labels:
    app: sentinel
spec:
  type: ClusterIP
  ports:
    - port: 8858
      targetPort: 8858
      name: web
    - port: 8719
      targetPort: 8719
      name: api
  selector:
    app: sentinel

```
```
kubectl apply -f svc -n nacos
```
### 4. 验证服务是否正常
```
kubectl get pod,svc -n nacos
kubectl logs -f sentinel-0 -n nacos
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618386839255-c4412e4b-2d7e-4b0b-a707-3ace77024e8b.png#align=left&display=inline&height=381&margin=%5Bobject%20Object%5D&name=image.png&originHeight=761&originWidth=1865&size=172275&status=done&style=none&width=932.5)
### 5. ingress 对外暴露sentinel dashboard
cat ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sentinel-http
  namespace: nacos
  annotations:
    kubernetes.io/ingress.class: traefik  
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: sentinel.saynaihe.com 
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: sentinel
            port:
              number: 8858
```
输入 configmap中设置的用户名密码
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618387004824-3327ec98-9e90-45c9-9767-1e583b031e8b.png#align=left&display=inline&height=406&margin=%5Bobject%20Object%5D&name=image.png&originHeight=811&originWidth=1600&size=66640&status=done&style=none&width=800)
进入控制台：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618387026793-3bbce256-2dff-41be-b3ed-2c3b70d302b7.png#align=left&display=inline&height=466&margin=%5Bobject%20Object%5D&name=image.png&originHeight=932&originWidth=1514&size=168401&status=done&style=none&width=757)
实时监控，请求链路  流控规则和降级规则这几个名词个人就很喜欢的样子.....后面再去研究下使用。


