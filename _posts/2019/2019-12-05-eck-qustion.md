---
layout: post
title: "2019-12-05-eck-qustion"
date: "2019-12-05 10:00:00"
category: "elastic-oparator"
tags:  "elastic-oparator Elastic Cloud on Kubernetes (ECK)"
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
> elastic-oparator搭建完成eck, kibana管理界面汉化和建立新用户权限设置。
> 
> 初始化环境参考https://duiniwukenaihe.github.io/2019/10/21/k8s-efk/。然后默认把7.4的镜像修改成7.5.0直接升级了一下


## 1. kibana汉化
 ```bash
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1beta1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic-system
spec:
  version: 7.5.0
  count: 1
  elasticsearchRef:
    name: "elastic"
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
> 这样就可以支持中文了

![chinese](/assets/images/efk/chinese.png)

## kibana创建新用户
> 按照官方教程 创建后貌似用户是无法登陆的，New user can’t login kibana。可参考https://discuss.elastic.co/t/new-user-cant-login-kibana/204810。

 ```bash
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1beta1
kind: Elasticsearch
metadata:
  name: elastic
  namespace: elastic-system
spec:
  version: 7.5.0
  nodeSets:
  - name: elastic
    count: 3
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
      xpack.security.authc.realms:
        native:
          native1: 
            order: 1
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 30Gi
        storageClassName: rook-ceph-block
EOF
 ```
 ```bash
增加了：
      xpack.security.authc.realms:
        native:
          native1: 
            order: 1
 ```
![user](/assets/images/efk/user.png)
>其他参考官方文档即可。