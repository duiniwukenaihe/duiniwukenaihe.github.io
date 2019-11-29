---
layout: post
title: "2019-11-29-k8s-helm-install-postgresql-sonarr"
date: "2019-11-29 10:00:00"
category: kubernetes
tags:  kubernetes  helm postgresql sonarqube
author: duiniwukenaihe
---
* content
{:toc}

 



集群配置：
初始集群环境kubeadm 1.16.1

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
> 正常来说helm玩的好应该是直接安装的但是玩的不太好 ，postgresql 和sonarqube分成两部安装的。变量各种用的不熟悉，安装后sonarqube报错什么的， 就分成两步安装了

# 开始安装

## 1. git clone chart库
> git clone https://github.com/helm/charts

## 2. helm 安装postgresql

> cd charts/stable/sonarqube/postgresql
> 
修改 values.yaml，就设置了了用户密码和存储storageClass。

 ```bash
postgresqlPassword: qmVy5wubfmcekZy3

storageClass: "rook-ceph-block"
 ```

![sonar1.png](/assets/images/sonar/sonar1.png)
![sonar2.png](/assets/images/sonar/sonar2.png)

 ```bash
helm install --name sonar-postgresql -f values.yaml . --namespace kube-ops
 ```

![sonar3.png](/assets/images/sonar/sonar3.png)

> 进入postgresql 创建sonar数据库

 ```bash
export POSTGRES_PASSWORD=$(kubectl get secret --namespace kube-ops sonar-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)

kubectl run sonar-postgresql-client --rm --tty -i --restart='Never' --namespace kube-ops --image docker.io/bitnami/postgresql:11.6.0-debian-9-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host sonar-postgresql -U postgres -d postgres -p 5432

postgres=# CREATE DATABASE sonar；
 ```
> kubectl delete pods sonar-postgresql-client -n kube-ops 客户端就删除了就是看helm安装的输出试了一下呢。

## 3. helm 安装sonar


 ```bash
cd ../sonarqube
rm -rf requirements.yaml
# 不删除的话还要检查依赖charts目录下放postgresql的 chart目录。放上试了几次没有整明白，就分开整了
修改values.yaml
#配置storageclass
  storageClass: rook-ceph-block
  accessMode: ReadWriteOnce
  size: 10Gi
#配置postgresql
postgresql:
  # Enable to deploy the PostgreSQL chart
  enabled: false  
  # To use an external PostgreSQL instance, set enabled to false and uncomment
  # the line below:
  postgresServer: "sonar-postgresql"
  # To use an external secret for the password for an external PostgreSQL
  # instance, set enabled to false and provide the name of the secret on the
  # line below:
  # postgresPasswordSecret: ""
  postgresUser: "postgres"
  postgresPassword: "qmVy5wubfmcekZy3"
  postgresDatabase: "sonar"
  # Specify the TCP port that PostgreSQL should use
  service:
    port: 5432

 ```
> helm install --name sonar -f values.yaml . --namespace kube-ops

 ```bash
kubectl get pods -n kube-ops

kubectl get svc -n kube-ops
 ```
![sonar4.png](/assets/images/sonar/sonar4.png)
![sonar5.png](/assets/images/sonar/sonar5.png)
![sonar6.png](/assets/images/sonar/sonar6.png)
## 4. treafik代理 sonar
 ``` bash
cat <<EOF > sonarqube-https.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: sonar-sonarqube-https
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(\`sonarqube.sainaihe.com\`)
      kind: Rule
      services:
        - name: sonar-sonarqube
          port: 9000
EOF
kubectl apply -f sonarqube-https.yaml
 ```
![sonar7.png](/assets/images/sonar/sonar7.png)
## 5. 登录sonar 修改密码设置语言包
![sonar8.png](/assets/images/sonar/sonar8.png)
![sonar9.png](/assets/images/sonar/sonar9.png)
![sonar10.png](/assets/images/sonar/sonar10.png)
> ok安装完成了，有时间整下和jenkins的结合跑个例子。