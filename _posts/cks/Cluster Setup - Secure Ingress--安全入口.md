## 前言
**ingress objects with security control             一个具有安全控制的入口对象**
**what is ingress？                           什么是入口**
**setup an ingress with services      使用服务设置入口**
**secure an ingress with tls     使用tls保护入口**
**参见官方文档**
[**https://kubernetes.io/docs/concepts/services-networking/ingress/**](https://kubernetes.io/docs/concepts/services-networking/ingress/)
**一个API对象，用于管理对集群中服务的外部访问，通常是HTTP。**
**入口可以提供负载平衡，SSL和基于名称的虚拟主机。**


## 1. 什么是ingress？
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615463887907-7b47c066-5f98-4514-bd03-2912cc1d3aa7.png#align=left&display=inline&height=416&margin=%5Bobject%20Object%5D&name=image.png&originHeight=416&originWidth=835&size=61753&status=done&style=none&width=835)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611146705671-26a28a5c-d082-4d82-8e9b-4a3a5fa2f0ec.png#align=left&display=inline&height=472&margin=%5Bobject%20Object%5D&name=image.png&originHeight=472&originWidth=869&size=77783&status=done&style=none&width=869)
一般的外部访问应用的流程

1. 通过yaml等资源创建一个nginx pod应用（也可以是其他应用，用nginx镜像就因为简单）
1. 外部用户通过LoadBalancer（负载均衡）映射到NodePort（主机暴露的3000-32767任一端口）通过ClusterIP（服务在集群内的ip)到pod应用暴露的端口。



metallb裸金属的代理方式 还有traefik kong  istio等是怎么玩的？也没有深入研究下traefik。个人使用就是用腾讯云的slb负载均衡使用tcp的方式绑定了work节点的80  443 Nodeport 然后service都是用了ClusterIP的方式。至于端口没有使用http https的方式是因为slb服务只能挂载一个证书。域名比较乱。为了方便管理，证书就使用在集群内用secret的方式管理挂载了。
kubernetes的网络通信有时间要深入研究一下了。自己太浮躁掌握的太浅。正巧今天看到了倪鹏飞大佬分享的如何快速掌握kubernetes网络，可以有时间看下：[https://mp.weixin.qq.com/s/Tq1dq57Y0FPgzwPxzixHoA](https://mp.weixin.qq.com/s/Tq1dq57Y0FPgzwPxzixHoA)。


## 2 . 举一个例子简单http的例子


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611146760228-dcaa223a-8970-44ef-9ef2-9e96e0338c2f.png#align=left&display=inline&height=507&margin=%5Bobject%20Object%5D&name=image.png&originHeight=507&originWidth=880&size=142244&status=done&style=none&width=880)
注： service1 service2就用默认的nginx  apache镜像去区分了。index.html毕竟不一样。很容易区分出来部署效果。
### 2.1 创建ingress-nginx入口
```html
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.40.2/deploy/static/provider/baremetal/deploy.yaml
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615519783046-4d34042a-3b9a-4a1b-8da5-a849bfbf73ac.png#align=left&display=inline&height=615&margin=%5Bobject%20Object%5D&name=image.png&originHeight=615&originWidth=1597&size=102091&status=done&style=none&width=1597)
嗯！镜像库被墙了，国外服务器下载了然后修改了标签上传到了自己的harbor仓库，然后直接修改deployment中image的标签。当然了我在work节点直接docker pull下了镜像。对于私有仓库正常的是建立一个秘钥secret在namespace,然后配置文件也添加上有pull镜像权限的secret的方式进行部署。
```html
kubectl edit deployment ingress-nginx-controller -n ingress-nginx
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615519592911-9d5b4acf-4d4a-4277-b57c-a640c12579e8.png#align=left&display=inline&height=493&margin=%5Bobject%20Object%5D&name=image.png&originHeight=493&originWidth=883&size=41918&status=done&style=none&width=883)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615530862669-995c2ecc-c2b5-4d41-803b-dc8301c748d3.png#align=left&display=inline&height=317&margin=%5Bobject%20Object%5D&name=image.png&originHeight=317&originWidth=1039&size=36464&status=done&style=none&width=1039)
OK ,访问master节点IP+nodeport 与work节点IP+nodeport都是可以正常返回的


### 2.2. 创建pod service 以及ingress入口
```html
kubectl run pods1 --image=nginx -n ingress
kubectl run pods2 --image=httpd -n ingress
kubectl expose pod pods1 --port 80 --name service1 -n ingress
kubectl expose pod pods2 --port 80 --name service2 -n ingress
```
注：随便编排在那一个命名空间都可以的，没有再default空间部署是因为我保留了再default空间的network policy。不想删除了，就建立一个新的空间没有网络规则的空间演示了。另外，这里是不是也可以玩下网络规则呢？题外话了哈哈。


创建ingress入口文件
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615530029987-73696c69-94ae-46c8-aeb9-dcbfe9019c89.png#align=left&display=inline&height=531&margin=%5Bobject%20Object%5D&name=image.png&originHeight=531&originWidth=921&size=34669&status=done&style=none&width=921)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615530150421-05e7e1f9-9375-46df-b7b6-7e4abde2e02c.png#align=left&display=inline&height=346&margin=%5Bobject%20Object%5D&name=image.png&originHeight=346&originWidth=1011&size=55987&status=done&style=none&width=1011)
### 2.3. 测试部署结果
```html
curl 10.0.4.14:31279/service1
curl 10.0.4.14:31279/service2
```
两个service返回不一样的 一个nginx  一个apache简单测试通过。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615532279483-bcdbbf38-f2ab-4b9a-baef-c0edcc2bff83.png#align=left&display=inline&height=486&margin=%5Bobject%20Object%5D&name=image.png&originHeight=486&originWidth=1104&size=49373&status=done&style=none&width=1104)
## 3. 创建一个https的例子


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615553846514-07e19425-84a7-4e7b-86cc-e20f55fa72ed.png#align=left&display=inline&height=292&margin=%5Bobject%20Object%5D&name=image.png&originHeight=583&originWidth=1114&size=246150&status=done&style=none&width=557)
### 3.1 首先确认一下https对外映射的端口是30437
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615554147216-ced419b1-fd99-42ef-b818-f9b19fca70bb.png#align=left&display=inline&height=53&margin=%5Bobject%20Object%5D&name=image.png&originHeight=105&originWidth=1080&size=14343&status=done&style=none&width=540)
### 3.2 关键一下直接访问是什么结果。以及熟悉下curl
```html
curl https://10.0.2.17:30437/service1
curl https://10.0.2.17:30437/service2
curl https://10.0.2.17:30437/service1 -k
curl https://10.0.2.17:30437/service2 -k
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615554309989-bb7fb6cc-c9fa-486e-892f-8058562abdd3.png#align=left&display=inline&height=332&margin=%5Bobject%20Object%5D&name=image.png&originHeight=663&originWidth=1030&size=74279&status=done&style=none&width=515)
嗯 -k忽略证书。
### 3.3 curl -kv 跟踪下证书
```html
curl https://10.0.2.17:30437/service1 -kv
curl https://10.0.2.17:30437/service2 -kv
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615554636597-485023c4-a76c-4198-988f-ac1d3dbf499c.png#align=left&display=inline&height=302&margin=%5Bobject%20Object%5D&name=image.png&originHeight=604&originWidth=1159&size=82606&status=done&style=none&width=579.5)
由上图可知ingress 入口默认的证书用的是kubernetes 自签名的通配符证书
### 3.4 创建自己的证书，并将证书以secret的方式挂载在相应命名空间
```html
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
kubernets create secret tls secure-ingress --cert=cert.pem   --key=key.pem -n ingress
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615555305044-1449d654-2d1b-4a6f-9b17-01ce3b22a1c8.png#align=left&display=inline&height=363&margin=%5Bobject%20Object%5D&name=image.png&originHeight=725&originWidth=1294&size=99264&status=done&style=none&width=647)
### 3.5 修改ingress配置文件
```html
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
      - secure-ingress.com
    secretName: secure-ingress
  rules:
  - host: secure-ingress.com
    http:
      paths:
      - path: /service1
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
      - path: /service2
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 80

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615555638263-6043c0bd-cf56-41e5-b953-9f9736d3fd0b.png#align=left&display=inline&height=272&margin=%5Bobject%20Object%5D&name=image.png&originHeight=543&originWidth=1201&size=45708&status=done&style=none&width=600.5)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615555704620-44174a4c-634a-4af0-928c-f3b352996bfc.png#align=left&display=inline&height=412&margin=%5Bobject%20Object%5D&name=image.png&originHeight=825&originWidth=1202&size=105917&status=done&style=none&width=601)
so  为什么还是kubernetes 自签名的通配符证书,创建自己的证书？因为配置文件里面的host是secure-ingress.com的域名方式。
### 3.6 正确的打开方式
#### 3.6.1host添加解析
```html
cat /etc/hosts
10.0.2.17 secure-ingress.com
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615555861143-8bd01ba2-5da5-438e-845d-497f8d3db95c.png#align=left&display=inline&height=405&margin=%5Bobject%20Object%5D&name=image.png&originHeight=809&originWidth=1137&size=108239&status=done&style=none&width=568.5)


#### 3.6.2 另外一种方式curl 的用法  resolve  
```html
curl https://secure-ingress.com:30437/service1 -kv --resolve secure-ingress.com:30437:10.0.2.17
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615556084823-2988fde1-3347-49dc-bdc5-5cc65aa68300.png#align=left&display=inline&height=354&margin=%5Bobject%20Object%5D&name=image.png&originHeight=708&originWidth=1263&size=100930&status=done&style=none&width=631.5)
OK简单的跑完。欠了两个基础知识，一个curl的各种玩法， 还有kubernetes的网络通信原理。有机会补上


