---
layout: post
title: Kubernetes 1.20.5 安装gitlab
date: 2021-04-01 01:00:00
category: kubernetes1.20
tags:  kubernetes gitlab 
author: duiniwukenaihe
---
* content
{:toc}
# 前言：
参照[https://www.yuque.com/duiniwukenaihe/ehb02i](https://www.yuque.com/duiniwukenaihe/ehb02i)内[https://www.yuque.com/duiniwukenaihe/ehb02i/qz49ev](https://www.yuque.com/duiniwukenaihe/ehb02i/qz49ev)之前文章。要完成kubernetes  devops工作流的完成。前面已经搭建了jenkins。gitlab代码仓库也是必不可缺少的。现在搞一下gitlab,关于helm前面也做了详细的讲述，这里略过了。另外之前gitlab版本没有中文版本可参照[https://hub.docker.com/r/twang2218/gitlab-ce-zh/](https://hub.docker.com/r/twang2218/gitlab-ce-zh/)  twang2218的汉化版本。现在的gitlab已经支持多语言了，可以略过。下面就开始安装gitlab。看了一眼helm的安装方式...文章较少。还是决定老老实实yaml方式安装了
# 1. 创建gitlab搭建过程中所需要的pvc
初步规划：存储storageclass是用的腾讯云开源的cbs-csi插件，由于最小值只能是10G，redis postgresql就设置为10G了。特意强调下 pvc指定namespace。昨天手贱安装kubesphere玩下了，结果发现他自带的Prometheus把我的pv,pvc抢占了....不知道这是cbs的坑还是自己搭建方式有问题。最后用户名密码一直错误。卸载了，不玩了......


cat gitlab-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pvc
  namespace: kube-ops
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: cbs-csi
```
cat gitlab-redis-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-redis-pvc
  namespace: kube-ops  
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs-csi
```
cat gitlab-pg-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pg-pvc
  namespace: kube-ops 
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs-csi
```
在当前目录下执行
```
kubectl apply -f .
```
![image.png](https://img-blog.csdnimg.cn/img_convert/f58d01b5c650285c6e7e9856e0b1cfdb.png#align=left&display=inline&height=205&margin=[objectObject]&name=image.png&originHeight=411&originWidth=1561&size=86589&status=done&style=none&width=780.5)
# 2. gitlab-redis搭建
注： 特意指定了namespace，否则执行kubectl  apply -f  yaml文件的时候经常会忘掉指定namespace
，claimName 修改为自己创建的pvc。

cat redis.yaml
```
## Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-redis
  namespace: kube-ops
  labels:
    name: gitlab-redis
spec:
  type: ClusterIP
  ports:
    - name: redis
      protocol: TCP
      port: 6379
      targetPort: redis
  selector:
    name: gitlab-redis
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-redis
  namespace: kube-ops
  labels:
    name: gitlab-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab-redis
  template:
    metadata:
      name: gitlab-redis
      labels:
        name: gitlab-redis
    spec:
      containers:
      - name: gitlab-redis
        image: 'sameersbn/redis:4.0.9-3'
        ports:
        - name: redis
          containerPort: 6379
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 1000m
            memory: 2Gi
        volumeMounts:
          - name: data
            mountPath: /var/lib/redis
        livenessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-redis-pvc
```
```
kubectl  apply -f redis.yaml
```
![image.png](https://img-blog.csdnimg.cn/img_convert/4a1f8e6e1f049fdebe68eb834f51f9e9.png#align=left&display=inline&height=241&margin=[objectObject]&name=image.png&originHeight=481&originWidth=1626&size=110455&status=done&style=none&width=813)
等待创建完成running。
# 3.gitlab-postgresql搭建
同redis 配置一样修改pg配置.

cat pg.yaml

```
## Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-postgresql
  namespace: kube-ops
  labels:
    name: gitlab-postgresql
spec:
  ports:
    - name: postgres
      protocol: TCP
      port: 5432
      targetPort: postgres
  selector:
    name: postgresql
  type: ClusterIP
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: postgresql
  namespace: kube-ops
  labels:
    name: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgresql
  template:
    metadata:
      name: postgresql
      labels:
        name: postgresql
    spec:
      containers:
      - name: postgresql
        image: sameersbn/postgresql:12-20200524
        ports:
        - name: postgres
          containerPort: 5432
        env:
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: admin@mydlq
        - name: DB_NAME
          value: gitlabhq_production
        - name: DB_EXTENSION
          value: 'pg_trgm,btree_gist'
        resources: 
          requests:
            cpu: 2
            memory: 2Gi
          limits:
            cpu: 2
            memory: 2Gi
        livenessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-pg-pvc
```
kubectl apply -f pg.yaml
![image.png](https://img-blog.csdnimg.cn/img_convert/bb8bbeeece5baa114dbb0b971ac222e6.png#align=left&display=inline&height=225&margin=[objectObject]&name=image.png&originHeight=449&originWidth=1451&size=102868&status=done&style=none&width=725.5)


# 4. gitlab deployment搭建
cat gitlab.yaml
```
## Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    name: gitlab
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
    - name: ssh
      protocol: TCP
      port: 22
  selector:
    name: gitlab
  type: ClusterIP
---
## Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab
  namespace: kube-ops
  labels:
    name: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        name: gitlab
    spec:
      containers:
      - name: gitlab
        image: 'sameersbn/gitlab:13.6.2'
        ports:
        - name: ssh
          containerPort: 22
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: GITLAB_TIMEZONE
          value: Beijing
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_ROOT_PASSWORD
          value: admin@mydlq
        - name: GITLAB_ROOT_EMAIL 
          value: 820042728@qq.com     
        - name: GITLAB_HOST           
          value: 'gitlab.saynaihe.com'
        - name: GITLAB_PORT        
          value: '80'                   
        - name: GITLAB_SSH_PORT   
          value: '22'
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: 'true'
        - name: GITLAB_NOTIFY_PUSHER
          value: 'false'
        - name: DB_TYPE             
          value: postgres
        - name: DB_HOST         
          value: gitlab-postgresql           
        - name: DB_PORT          
          value: '5432'
        - name: DB_USER        
          value: gitlab
        - name: DB_PASS         
          value: admin@mydlq
        - name: DB_NAME          
          value: gitlabhq_production
        - name: REDIS_HOST
          value: gitlab-redis              
        - name: REDIS_PORT      
          value: '6379'
        resources: 
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 2
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 300
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 30
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: data
          mountPath: /home/git/data
        - name: localtime
          mountPath: /etc/localtime
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-pvc
      - name: localtime
        hostPath:
          path: /etc/localtime
```
基本抄的豆丁大佬的文档。但是删掉了NodePort的方式。还是喜欢用ingress的代理方式。密码 用户名配置的可以安装自己的需求更改了。
![image.png](https://img-blog.csdnimg.cn/img_convert/99ac829325bdc057b5d9df4d1bd7d80e.png#align=left&display=inline&height=265&margin=[objectObject]&name=image.png&originHeight=530&originWidth=1507&size=119252&status=done&style=none&width=753.5)
等待running......
# 5. ingress配置
cat ingress.yaml
```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: gitlab-http
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`gitlab.saynaine.com`)
      kind: Rule
      services:
        - name: gitlab
          port: 80

```
kubectl apply -f ingress.yaml
访问 gitlab.saynaihe.com(域名仍然为虚构.)。都做了强制跳转了。故访问的伟http页面默认用户名root，密码是自己gitlab.yaml文件中设置的。（至于显示中文，是因为我的谷歌浏览器安装了中文翻译插件）
![image.png](https://img-blog.csdnimg.cn/img_convert/d068df87c4d2ac6479b01e50040be763.png#align=left&display=inline&height=385&margin=[objectObject]&name=image.png&originHeight=769&originWidth=1710&size=71096&status=done&style=none&width=855)
OK，登陆成功
# 6. 关闭用户注册，更改默认语言为中文。
![image.png](https://img-blog.csdnimg.cn/img_convert/e77353726451c64182ac8ba989bafe0b.png#align=left&display=inline&height=486&margin=[objectObject]&name=image.png&originHeight=972&originWidth=1641&size=115654&status=done&style=none&width=820.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/4363ed033a981c783f3509d28d9ae64e.png#align=left&display=inline&height=484&margin=[objectObject]&name=image.png&originHeight=967&originWidth=1496&size=214516&status=done&style=none&width=748)
![image.png](https://img-blog.csdnimg.cn/img_convert/8bd3027abc07471bd748c3b3fc1430d1.png#align=left&display=inline&height=489&margin=[objectObject]&name=image.png&originHeight=977&originWidth=1674&size=120209&status=done&style=none&width=837)
![image.png](https://img-blog.csdnimg.cn/img_convert/519229e164a360214f77b7270d54480c.png#align=left&display=inline&height=414&margin=[objectObject]&name=image.png&originHeight=828&originWidth=1652&size=70286&status=done&style=none&width=826)
基本安装完成。其他的用法以后慢慢研究....... 现在就是先把工具链安装整合起来。对了gitlab 登陆后记得更改用户名密码....增加个人安全意识是很有必要的。








