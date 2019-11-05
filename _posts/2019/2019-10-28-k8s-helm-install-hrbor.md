---
layout: post
title: "2019-10-28-k8s-helm-install-harbor"
date: "2019-10-29 10:00:00"
category: kubernetes
tags:  kubernetes  helm harbor
author: duiniwukenaihe
---
* content
{:toc}

 

# 描述背景：
注：记录各种常见问题

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

# 安装harbor

 ```bash
# 下载harbor仓库
git clone https://github.com/goharbor/harbor-helm
注：偶尔会下载了 部署不成功，如有此状况尽量选择稳定分支。
#修改配置文件values.yaml
代理使用了treafik,对外暴露模式没有使用loadBalancer和nodePort,选择了clusterIP，然后使用treafik代理，集群中安装了rook-ceph.存储storageClass: "rook-ceph-block"配置文件就修改了这两个配置。
  ```
![clusterIP.png](/assets/images/harbor/clusterIP.png)
![storageClass.png](/assets/images/harbor/storageClass.png)

# helm安装 harbor

 ```bash
helm install --name harbor -f values.yaml . --namespace kube-ops
# 等待pod running
kubectl get pods -n kube-ops -w  
如果pod一直pending 基本是pv,pvc的问题 查看下自己的storageclass ,pv,pvc配置
```

![svc.png](/assets/images/harbor/svc.png)

# treafik 代理harbor
 ```bash
kube-ops下创建http证书(ssl证书目录下执行)
kubectl create secret tls all-saynaihe-com --key=2_sainaihe.com.key --cert=1_saynaihe.com_bundle.crt -n kube-ops

cat <<EOF > harbor.saynaihe.com.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-https
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/`)
      kind: Rule
      services:
        - name: harbor-harbor-portal
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-api
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: harbor-harbor-core
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-service
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/service`)
      kind: Rule
      services:
        - name: harbor-harbor-core
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-v2
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/v2`)
      kind: Rule
      services:
        - name: harbor-harbor-core
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-chartrepo
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/chartrepo`)
      kind: Rule
      services:
        - name: harbor-harbor-core
          port: 80
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: harbor-c
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(`harbor.saynaihe.com`) && PathPrefix(`/c`)
      kind: Rule
      services:
        - name: harbor-harbor-core
          port: 80
EOF
kubectl apply -f harbor.saynaihe.com.yaml
登录https://traefik.lsaynaihe.com/dashboard/#/http/routers 查看,如下图：

```


![harbor.png](/assets/images/harbor/harbor.png)

# web登录harbor
``` bash
注：密码为配置文件中默认Harbor12345，在安装前可自定义修改，我选择了登录后自主修改：
OK安装完成，harbor还在摸索中，最喜欢的一个功能是同步其他仓库非常方便：
```
![harbor2.png](/assets/images/harbor/harbor2.png)

![harbor3.png](/assets/images/harbor/harbor3.png)

![harbor4.png](/assets/images/harbor/harbor4.png)