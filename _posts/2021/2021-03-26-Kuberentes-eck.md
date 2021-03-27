---
layout: post
title: Kuberentes 1.20.5搭建eck
date: 2021-03-26 18:00:00
category: kubernetes1.20
tags:  kubernetes eck
author: duiniwukenaihe
---
* content
{:toc}



# 前言：
kubernetes1.16版本的时候安装了elastic on kubernetes（ECK）1.0版本。存储用了local disk文档跑了一年多了。elasticsearch对应版本是7.6.2。现在已完成了kubernetes 1.20.5 containerd cilium hubble 环境的搭建（[https://blog.csdn.net/saynaihe/article/details/115187298](https://blog.csdn.net/saynaihe/article/details/115187298)）并且集成了cbs腾讯云块存储（[https://blog.csdn.net/saynaihe/article/details/115212770](https://blog.csdn.net/saynaihe/article/details/115212770)）。eck也更新到了1.5版本(我能说我前天安装的时候还是1.4.0吗.....还好我只是简单应用没有太复杂的变化无非版本变了....那就再来一遍吧)
最早部署的kubernetes1.16版本的eck安装方式[https://duiniwukenaihe.github.io/2019/10/21/k8s-efk/](https://duiniwukenaihe.github.io/2019/10/21/k8s-efk/)多年前搭建的eck1.0版本。
## 关于eck  
elastic cloud on kubernetes是一种operator的安装方式，很大程度上简化了应用的部署。同样的还有promtheus-operator。
可参照[https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html)官方文档的部署方式。
# 1. 在kubernete集群中部署ECK
## 1. 安装[自定义资源定义](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)和操作员及其RBAC规则：
kubectl apply -f https://download.elastic.co/downloads/eck/1.5.0/all-in-one.yaml
## 2. 监视操作日志：
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
**
-----------------------分隔符-----------------
我是直接把yaml下载到本地了。
```
###至于all-in-one.yaml.1后面的1可以忽略了哈哈，第二次加载文件加后缀了。
kubectl apply -f all-in-one.yaml
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator

```
![image.png](https://img-blog.csdnimg.cn/img_convert/5fb2293465854b031289a4ff8888428a.png#align=left&display=inline&height=284&margin=[objectObject]&name=image.png&originHeight=567&originWidth=1484&size=108969&status=done&style=none&width=742)
# 2. 部署elasticsearch集群
## 1. 定制化elasticsearch 镜像
增加s3插件，修改时区东八区，并添加腾讯云cos的秘钥，并重新打包elasticsearch镜像.
#### 1. DockerFile如下
```
FROM docker.elastic.co/elasticsearch/elasticsearch:7.12.0
ARG ACCESS_KEY=XXXXXXXXX
ARG SECRET_KEY=XXXXXXX
ARG ENDPOINT=cos.ap-shanghai.myqcloud.com
ARG ES_VERSION=7.12.0

ARG PACKAGES="net-tools lsof"
ENV allow_insecure_settings 'true'
RUN rm -rf /etc/localtime && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

RUN echo 'Asia/Shanghai' > /etc/timezone
RUN  if [ -n "${PACKAGES}" ]; then  yum install -y $PACKAGES && yum clean all && rm -rf /var/cache/yum; fi
RUN \
     /usr/share/elasticsearch/bin/elasticsearch-plugin install --batch repository-s3 && \
    /usr/share/elasticsearch/bin/elasticsearch-keystore create && \
	echo "XXXXXX"  | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin s3.client.default.access_key  && \
	echo "XXXXXX"  | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin s3.client.default.secret_key
```
#### 2. 打包镜像并上传到腾讯云镜像仓库，也可以是其他私有仓库
```
 docker build -t ccr.ccs.tencentyun.com/xxxx/elasticsearch:7.12.0 .
 docker push  ccr.ccs.tencentyun.com/xxxx/elasticsearch:7.12.0
```
![image.png](https://img-blog.csdnimg.cn/img_convert/02fbb6a5bcf9185f1492b127ab9435dd.png#align=left&display=inline&height=266&margin=[objectObject]&name=image.png&originHeight=531&originWidth=1067&size=61775&status=done&style=none&width=533.5)
## 2. 创建elasticsearch部署yaml文件，部署elasticsearch集群
修改了自己打包 image tag ，使用了腾讯云cbs csi块存储。并定义了部署的namespace，创建了namespace logging.


### 1. 创建部署elasticsearch应用的命名空间
```
kubectl create ns logging
```
```
cat <<EOF > elastic.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elastic
  namespace: logging
spec:
  version: 7.12.0
  image: ccr.ccs.tencentyun.com/XXXX/elasticsearch:7.12.0
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  nodeSets:
  - name: laya
    count: 3
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms2g -Xmx2g
          resources:
            requests:
              memory: 4Gi
              cpu: 0.5
            limits:
              memory: 4Gi
              cpu: 2
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        storageClassName: cbs-csi 
        resources:
          requests:
            storage: 200Gi
EOF          
```
![image.png](https://img-blog.csdnimg.cn/img_convert/fd20a41590032cc422d70903c2e84732.png#align=left&display=inline&height=345&margin=[objectObject]&name=image.png&originHeight=689&originWidth=1524&size=55319&status=done&style=none&width=762)
### 2. 部署yaml文件并查看应用部署状态
```
kubectl apply -f elastic.yaml
kubectl get elasticsearch -n logging
kubectl get elasticsearch -n logging
kubectl -n logging get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=elastic'
```
![image.png](https://img-blog.csdnimg.cn/img_convert/c7ff88271584c17440c1324984fed655.png#align=left&display=inline&height=171&margin=[objectObject]&name=image.png&originHeight=342&originWidth=1200&size=48886&status=done&style=none&width=600)
## 3. 获取elasticsearch凭据
```
kubectl -n logging get secret elastic-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```
![image.png](https://img-blog.csdnimg.cn/img_convert/73759106828be81b9173e0eaaedf60f6.png#align=left&display=inline&height=32&margin=[objectObject]&name=image.png&originHeight=63&originWidth=1206&size=8669&status=done&style=none&width=603)
## 4. 直接安装kibana了
修改了时区，和elasticsearch镜像一样都修改到了东八区，并将语言设置成了中文，关于selfSignedCertificate原因参照[https://www.elastic.co/guide/en/cloud-on-k8s/1.4/k8s-kibana-http-configuration.html](https://www.elastic.co/guide/en/cloud-on-k8s/1.4/k8s-kibana-http-configuration.html)。
```
cat <<EOF > kibana.yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: elastic
  namespace: logging
spec:
  version: 7.12.0
  image: docker.elastic.co/kibana/kibana:7.12.0
  count: 1
  elasticsearchRef:
    name: elastic
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  podTemplate:
    spec:
      containers:
      - name: kibana
        env:
        - name: I18N_LOCALE
          value: zh-CN
        resources:
          requests:
            memory: 1Gi
          limits:
            memory: 2Gi
        volumeMounts:
        - name: timezone-volume
          mountPath: /etc/localtime
          readOnly: true
      volumes:
      - name: timezone-volume
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
EOF          
```
```
kubectl apply kibana.yaml
```
## 5. 对外映射kibana 服务
对外暴露都是用traefik https 代理，命名空间添加tls secret。绑定内部kibana service.然后外部slb udp代理443端口。但是现在腾讯云slb可以挂载多个证书了，就把这层剥离了，直接http方式到80端口 。然后https 证书 都在slb负载均衡代理了。这样省心了证书的管理，还有一点是可以在slb层直接收集接入层日志到cos。并可使用腾讯云自有的日志服务。
```
cat <<EOF > ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kibana-kb-http
  namespace: logging
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`kibana.XXXXX.com`)
      kind: Rule
      services:
        - name: elastic-kb-http
          port: 5601
EOF          
kubectl apply -f ingress.yaml

```
输入 用户名elastic  密码为上面获取的elasticsearch的凭据，进入管理页面。新界面很是酷炫
![image.png](https://img-blog.csdnimg.cn/img_convert/2381c3d8a5cba6ab10e32dd8c359d580.png#align=left&display=inline&height=460&margin=[objectObject]&name=image.png&originHeight=919&originWidth=1825&size=126876&status=done&style=none&width=912.5)
## 6. now  要添加快照仓库了
![image.png](https://img-blog.csdnimg.cn/img_convert/3b18731d0903867384ec0d1ac749f34f.png#align=left&display=inline&height=408&margin=[objectObject]&name=image.png&originHeight=815&originWidth=1790&size=81256&status=done&style=none&width=895)
创建快照仓库跟S3方式是一样的，具体的可以参考[https://blog.csdn.net/ypc123ypc/article/details/87860583](https://blog.csdn.net/ypc123ypc/article/details/87860583)这篇博文
```
PUT _snapshot/esbackup
{
  "type": "s3",
  "settings": {
    "endpoint":"cos.ap-shanghai.myqcloud.com",
    "region": "ap-shanghai",
    "compress" : "true",
    "bucket": "elastic-XXXXXXX"
  }
}
```


![image.png](https://img-blog.csdnimg.cn/img_convert/0e93cc9544d54109af90a7ec43255744.png#align=left&display=inline&height=95&margin=[objectObject]&name=image.png&originHeight=190&originWidth=1408&size=21063&status=done&style=none&width=704)
OK 进行验证快照仓库是否添加成功
![image.png](https://img-blog.csdnimg.cn/img_convert/34145373e57048f3adf8eb92d90dfed5.png#align=left&display=inline&height=448&margin=[objectObject]&name=image.png&originHeight=895&originWidth=1318&size=81891&status=done&style=none&width=659)
![image.png](https://img-blog.csdnimg.cn/img_convert/ddb99648b1b8f497226a76fe9bbd0440.png#align=left&display=inline&height=400&margin=[objectObject]&name=image.png&originHeight=800&originWidth=1834&size=174926&status=done&style=none&width=917)
还原一个试试？
![image.png](https://img-blog.csdnimg.cn/img_convert/4e8d273bba0836b6c765eb081d64d4aa.png#align=left&display=inline&height=406&margin=[objectObject]&name=image.png&originHeight=811&originWidth=1689&size=93695&status=done&style=none&width=844.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/02437897ce8744f9c1bdd5d89f792aca.png#align=left&display=inline&height=395&margin=[objectObject]&name=image.png&originHeight=789&originWidth=1665&size=93193&status=done&style=none&width=832.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/d024ae68067c6b86a906bdeb630b4b64.png#align=left&display=inline&height=321&margin=[objectObject]&name=image.png&originHeight=642&originWidth=1407&size=46642&status=done&style=none&width=703.5)
可还行？等待变绿
![image.png](https://img-blog.csdnimg.cn/img_convert/cd05dd98b895040dd570f460dddd936e.png#align=left&display=inline&height=334&margin=[objectObject]&name=image.png&originHeight=667&originWidth=1505&size=67884&status=done&style=none&width=752.5)
基本完成。正常可以使用了。使用过程中还有很多注意的。关键还是集群的设计规划。数据的预估增长还有报警。下次有时间列一下Elastalert在kubernetes中的部署应用。




















