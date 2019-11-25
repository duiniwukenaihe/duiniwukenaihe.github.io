---
layout: "post"
title: "k8s-install-jenkins"
date:   "2019-11-19 14:00:33"
category: "kubernetes"
tags:  "kubernetes1.16 jenkins"
author: duiniwukenaihe
---
* content
{:toc}

 

# 描述背景：
注：kubernetes基本环境搭建完成，存储rook-ceph，rbd方式。代码仓库gitlab,容器仓库harbor,监控prometheus，负载方式都用了内部clusterip然后 traefik代理的方式。为了完善工具链，容器中搭建jenkins工具。

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


# 安装jenkins
> 1.  建立命名空间
 ```bash 
kubectl create namespace kube-ops
注：后续所有工具类应用程序都创建在此命名空间内。
 ```
---
> 2. 创建ServiceAccount & ClusterRoleBinding

 ```bash
注：都是用的默认的，权限的管理还没有深入进行学习下。

cat <<EOF > rabc.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins2
  namespace: kube-ops
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins2
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins2
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins2
subjects:
  - kind: ServiceAccount
    name: jenkins2
    namespace: kube-ops
EOF
kubectl apply -f rabc.yaml
 ```

> 3.deployment jenkins

 ```bash
cat <<EOF > jenkins.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: opspvc
  labels:
    app: jenkins2
  namespace: kube-ops
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins2
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: jenkins2
  template:
    metadata:
      labels:
        app: jenkins2
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins2
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          subPath: jenkins2
          mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: opspvc

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins2
  namespace: kube-ops
  labels:
    app: jenkins2
spec:
  selector:
    app: jenkins2
  ports:
  - name: web
    port: 8080
    targetPort: web
  - name: agent
    port: 50000
    targetPort: agent
EOF
kubectl apply -f jenkins.yaml
注： kubernetes 1.16 取消了extensions/v1beta1 api，使用apps/v1。
 ```
![deployment.png](/assets/images/jenkins/deployment.png)
> 4.traefik对外暴露服务
 ``` bash
cat <<EOF > ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: jenkins2-https
spec:
  entryPoints:
    - websecure
  tls:
    secretName: all-saynaihe-com
  routes:
    - match: Host(\`jenkins.sainaihe.com\`)
      kind: Rule
      services:
        - name: jenkins2
          port: 8080
EOF
kubectl apply -f ingress.yaml
 ```
![ingressroute.png](/assets/images/jenkins/ingressroute.png)
![traefik.png](/assets/images/jenkins/traefik.png)
## 登陆jenkins。初始化配置
> 1. 访问 https://jenkins.saynaihe.com,出现：
![jenkins1.png](/assets/images/jenkins/jenkins1.png)
> 获取初始密码
 ``` bash
kubectl exec -it jenkins2-9f55b98b6-xtffb cat /var/jenkins_home/secrets/initialAdminPassword -n kube-ops
 ```
![jenkins2.png](/assets/images/jenkins/jenkins2.png)
> 2. 设置管理员账号密码，进入登陆界面
![jenkins3.png](/assets/images/jenkins/jenkins3.png)
> 3. 安装插件，安装了中文插件，pipeline，gitlab,git,github 参数化插件等，看个人需要安装吧。
![jenkins4.png](/assets/images/jenkins/jenkins4.png)
![jenkins5.png](/assets/images/jenkins/jenkins5.png)
> 注：因为国外源不稳定 国内有其他备用源可以切换比如清华的源。jenkins中文社区有篇文章：https://mp.weixin.qq.com/s/rqx93WI0UEvzqaFrt84i8A可以参考。