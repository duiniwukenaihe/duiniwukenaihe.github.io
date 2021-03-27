---
layout: post
title: Kubernetes 1.20.5 安装traefik在腾讯云下的实践
date: 2021-03-26 17:00:00
category: kubernetes1.20
tags:  kubernetes traefik
author: duiniwukenaihe
---
* content
{:toc}
# 背景
个人使用traefik有差不多1-2年时间，kubernetes ingress controller 代理有很多种方式 例如 ingress-nginx  kong  istio 等等。个人比较习惯traefik。从19年就开始使用。最早使用traefik 不直接使用腾讯云公有云的slb是因为当时slb不能挂载多个证书，而我kubernetes的自建集群实在不想挂载多个slb.就偷懒用了slb  udp绑定运行traefik节点的 80  443端口。证书tls的secret 直接挂载在traefik代理层上面。hsts  http跳转https的特性都配置在了traefik代理层上面。应用比较少。qps也没有那么高，这样的简单应用就满足了我的需求了
## 关于traefik的结缘
最早接触traefik是Google上面看ingress controller 找到的 然后再阳明大佬的博客看到了traefik的实践[https://www.qikqiak.com/post/traefik2-ga/](https://www.qikqiak.com/post/traefik2-ga/)，还有超级小豆丁的博客[http://www.mydlq.club/article/41](http://www.mydlq.club/article/41) 。两位大佬的博客是kubernetes初学者的宝藏博客值得收藏拜读。
顺便吐个糟，用的traefik2.4版本... 抄的豆丁大佬的..[http://www.mydlq.club/article/107/](http://www.mydlq.club/article/107/)哈哈哈
## ingress controller对比：
参照[https://zhuanlan.zhihu.com/p/109458069](https://zhuanlan.zhihu.com/p/109458069)。
![](https://img-blog.csdnimg.cn/img_convert/b31233969a25874c0e348aea86bfba47.png#align=left&display=inline&height=742&margin=[objectObject]&originHeight=742&originWidth=1600&size=0&status=done&style=none&width=1600)
# 1. Kubernetes Gateway API 
v2.4版本的改变（在 Traefik v2.4 版本中增加了对 Kubernetes Gateway API 的支持）一下部分抄自豆丁大佬与官方文档[https://gateway-api.sigs.k8s.io/](https://gateway-api.sigs.k8s.io/)。
## 1、Gateway API 是什么
Gateway API是由[SIG-NETWORK](https://github.com/kubernetes/community/tree/master/sig-network) 社区管理的一个开源项目。它是在Kubernetes中对服务网络建模的资源的集合。这些资源- ，`GatewayClass`，`Gateway`，`HTTPRoute`， `TCPRoute`，`Service`等-旨在通过表现力，可扩展和面向角色由很多供应商实现的，并具有广泛的行业支持接口演进Kubernetes服务网络。
_注意：此项目以前被称为“服务API”，直到2021年2月被重命名为“_Gateway API _”。_
## 2、Gateway API 的目标
Gateway API 旨在通过提供可表达的，可扩展的，面向角色的接口来改善服务网络，这些接口已由许多供应商实施并获得了广泛的行业支持。
网关 API 是 API 资源（服务、网关类、网关、HTTPRoute、TCPRoute等）的集合。这些资源共同为各种网络用例建模。
![](https://img-blog.csdnimg.cn/img_convert/45e617bd8f628b4198fdc35ff75d04b3.png#align=left&display=inline&height=1418&margin=[objectObject]&originHeight=1418&originWidth=2414&size=0&status=done&style=none&width=2414)
Gateway API 如何根据 Ingress 等当前标准进行改进？

- 以下设计目标驱动了Gateway API的概念。这些证明了Gateway如何旨在改进Ingress等当前标准。
- **面向角色**-网关由API资源组成，这些API资源对使用和配置Kubernetes服务网络的组织角色进行建模。
- **便携式**-这不是改进，而是应该保持不变。就像Ingress是具有[许多实现](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)的通用规范一样 ，Gateway API也被设计为受许多实现支持的可移植规范。
- **富有表现力**-网关API资源支持核心功能，例如基于标头的匹配，流量加权以及其他只能通过自定义批注在Ingress中实现的功能。
- **可扩展**-网关API允许在API的各个层上链接自定义资源。这样就可以在API结构内的适当位置进行精细的自定义。

其他一些值得注意的功能包括：

- **GatewayClasses** -GatewayClasses形式化负载平衡实现的类型。这些类使用户可以轻松，明确地了解通过Kubernetes资源模型可以使用的功能。
- **共享网关和跨命名空间支持**-通过允许独立的Route资源绑定到同一网关，它们可以共享负载平衡器和VIP。这允许团队（甚至跨命名空间）在没有直接协调的情况下安全地共享基础结构。
- **类型化路由和类型化后端**-网关API支持类型化路由资源以及不同类型的后端。这使API可以灵活地支持各种协议（例如HTTP和gRPC）和各种后端目标（例如Kubernetes Services，存储桶或函数）。

如果想了解更多内容，可以访问 [Kubernetes Gateway API 文档](https://gateway-api.sigs.k8s.io/) 。
# 2. traefik on kubernetes实践
部署玩Traefik 应用后，创建外部访问 Kubernetes 内部应用的路由规则，才能从外部访问kubernetes内部应用。Traefik 目前支持三种方式创建路由规则方式，一种是创建 Traefik 自定义 `Kubernetes CRD` 资源，另一种是创建 `Kubernetes Ingress` 资源，还有就是 v2.4 版本对 Kubernetes 扩展 API `Kubernetes Gateway API` 适配的一种方式，创建 `GatewayClass`、`Gateway` 与 `HTTPRoute` 资源
**注意：这里 Traefik 是部署在 kube-system namespace 下，如果不想部署到配置的 namespace，需要修改下面部署文件中的 namespace 参数。当然了也可以新建一个单独的namespace去部署traefik**
## 1. 创建CRD
参照[https://doc.traefik.io/traefik/reference/dynamic-configuration/kubernetes-crd/](https://doc.traefik.io/traefik/reference/dynamic-configuration/kubernetes-crd/)


traefik-crd.yaml
```
cat <<EOF >  traefik-crd.yaml 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: serverstransports.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: ServersTransport
    plural: serverstransports
    singular: serverstransport
  scope: Namespaced
EOF
kubectl apply -f traefik-crd.yaml
  
```


## 2. 创建RBAC权限
```
cat <<EOF > traefik-rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
      - serverstransports
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
EOF
 kubectl apply -f traefik-rbac.yaml 
```
## 3. 创建 Traefik 配置文件
#号后为注释，跟2.X前几个版本一样。增加了kubernetesIngress  kubernetesGateway两种路由方式，过去只部署了CRD的方式。
```
cat <<EOF > traefik-config.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: traefik-config
  namespace: kube-system
data:
  traefik.yaml: |-
    ping: ""                    ## 启用 Ping
    serversTransport:
      insecureSkipVerify: true  ## Traefik 忽略验证代理服务的 TLS 证书
    api:
      insecure: true            ## 允许 HTTP 方式访问 API
      dashboard: true           ## 启用 Dashboard
      debug: false              ## 启用 Debug 调试模式
    metrics:
      prometheus: ""            ## 配置 Prometheus 监控指标数据，并使用默认配置
    entryPoints:
      web:
        address: ":80"          ## 配置 80 端口，并设置入口名称为 web
      websecure:
        address: ":443"         ## 配置 443 端口，并设置入口名称为 websecure
    providers:
      kubernetesCRD: ""         ## 启用 Kubernetes CRD 方式来配置路由规则
      kubernetesIngress: ""     ## 启用 Kubernetes Ingress 方式来配置路由规则
      kubernetesGateway: ""     ## 启用 Kubernetes Gateway API
    experimental:               
      kubernetesGateway: true   ## 允许使用 Kubernetes Gateway API
    log:
      filePath: ""              ## 设置调试日志文件存储路径，如果为空则输出到控制台
      level: error              ## 设置调试日志级别
      format: json              ## 设置调试日志格式
    accessLog:
      filePath: ""              ## 设置访问日志文件存储路径，如果为空则输出到控制台
      format: json              ## 设置访问调试日志格式
      bufferingSize: 0          ## 设置访问日志缓存行数
      filters:
        #statusCodes: ["200"]   ## 设置只保留指定状态码范围内的访问日志
        retryAttempts: true     ## 设置代理访问重试失败时，保留访问日志
        minDuration: 20         ## 设置保留请求时间超过指定持续时间的访问日志
      fields:                   ## 设置访问日志中的字段是否保留（keep 保留、drop 不保留）
        defaultMode: keep       ## 设置默认保留访问日志字段
        names:                  ## 针对访问日志特别字段特别配置保留模式
          ClientUsername: drop  
        headers:                ## 设置 Header 中字段是否保留
          defaultMode: keep     ## 设置默认保留 Header 中字段
          names:                ## 针对 Header 中特别字段特别配置保留模式
            User-Agent: redact
            Authorization: drop
            Content-Type: keep
    #tracing:                     ## 链路追踪配置,支持 zipkin、datadog、jaeger、instana、haystack 等 
    #  serviceName:               ## 设置服务名称（在链路追踪端收集后显示的服务名）
    #  zipkin:                    ## zipkin配置
    #    sameSpan: true           ## 是否启用 Zipkin SameSpan RPC 类型追踪方式
    #    id128Bit: true           ## 是否启用 Zipkin 128bit 的跟踪 ID
    #    sampleRate: 0.1          ## 设置链路日志采样率（可以配置0.0到1.0之间的值）
    #    httpEndpoint: http://localhost:9411/api/v2/spans     ## 配置 Zipkin Server 端点
EOF
kubectl apply -f traefik-config.yaml
```
## 4. 设置节点label标签
Traefix 采用 DaemonSet方式构建，在需要安装的节点上面打上标签，这里在三个work节点都安装上了默认：
```
kubectl label nodes {sh-work-01,sh-work-02,sh-work-02} IngressProxy=true
kubectl get nodes --show-labels
```
![image.png](https://img-blog.csdnimg.cn/img_convert/a0f87f352f1ca682216e7050a3be3161.png#align=left&display=inline&height=339&margin=[objectObject]&name=image.png&originHeight=677&originWidth=1188&size=126050&status=done&style=none&width=594)
![image.png](https://img-blog.csdnimg.cn/img_convert/4ada6304aa4835f6c0e6f2a2b13161b8.png#align=left&display=inline&height=120&margin=[objectObject]&name=image.png&originHeight=239&originWidth=1563&size=57181&status=done&style=none&width=781.5)
注意：如果想删除标签，可以使用 **kubectl label nodes k8s-node-03 IngressProxy- **命令。哈哈哈偶尔需要去掉标签，不调度。
## 5、安装 Kubernetes Gateway CRD 资源
由于目前 Kubernetes 集群上默认没有安装 Service APIs，我们需要提前安装 Gateway API 的 CRD 资源，需要确保在 Traefik 安装之前启用 Service APIs 资源。
```
kubectl apply -k "github.com/kubernetes-sigs/service-apis/config/crd?ref=v0.2.0"
```
不过由于github网络问题，基本无法安装的。我是直接把github上包下载到本地采用本地安装的方式安装
进入
![image.png](https://img-blog.csdnimg.cn/img_convert/36138353fc0d487898dee680e5d06fd1.png#align=left&display=inline&height=381&margin=[objectObject]&name=image.png&originHeight=762&originWidth=1672&size=114244&status=done&style=none&width=836)
进入base目录直接全部安装:
```
kubectl  apply  -f .
```
![image.png](https://img-blog.csdnimg.cn/img_convert/3c4d199cf29cd3b486c429349b400ec6.png#align=left&display=inline&height=309&margin=[objectObject]&name=image.png&originHeight=618&originWidth=1446&size=101285&status=done&style=none&width=723)
##  6. Kubernetes 部署 Traefik
其实我就可以忽略443了....因为我想在[slb](https://console.cloud.tencent.com/clb)  哦 对也叫[clb](https://console.cloud.tencent.com/clb).直接做限制。对外只保留80端口。
```
cat <<EOF > traefik-deploy.yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik
  namespace： kube-system
spec:
  ports:
    - name: web
      port: 80
    - name: websecure
      port: 443
    - name: admin
      port: 8080
  selector:
    app: traefik
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace： kube-system
  name: traefik-ingress-controller
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      name: traefik
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 1
      containers:
        - image: ccr.ccs.tencentyun.com/XXXX/traefik:v2.4.3 
          name: traefik-ingress-lb
          ports:
            - name: web
              containerPort: 80
              hostPort: 80     
            - name: websecure
              containerPort: 443
              hostPort: 443        
            - name: admin
              containerPort: 8080  
          resources:
            limits:
              cpu: 2000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          args:
            - --configfile=/config/traefik.yaml
          volumeMounts:
            - mountPath: "/config"
              name: "config"
          readinessProbe:
            httpGet:
              path: /ping
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          livenessProbe:
            httpGet:
              path: /ping
              port: 8080
            failureThreshold: 3
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5    
      volumes:
        - name: config
          configMap:
            name: traefik-config 
      tolerations:              ## 设置容忍所有污点，防止节点被设置污点
        - operator: "Exists"
      nodeSelector:             ## 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true
EOF
kubectl apply -f traefik-deploy.yaml 
```
kubectl get pods -n kube-system 验证
![image.png](https://img-blog.csdnimg.cn/img_convert/75325eac6c89406f5abbb9c6352323c9.png#align=left&display=inline&height=342&margin=[objectObject]&name=image.png&originHeight=683&originWidth=1009&size=91180&status=done&style=none&width=504.5)
# 3. 配置路由规则，与腾讯云clb整合
## 1. slb 绑定traefik  http端口
关于腾讯云负载均衡 slb  or  clb可以参照文档[https://cloud.tencent.com/document/product/214](https://cloud.tencent.com/document/product/214)了解。过去使用slb用的tcp代理方式有一下原因：

1. 过去的腾讯云slb不支持一个负载均衡挂载多个证书，个人不想启用多个slb绑定。
1. 在slb上面配置域名比较麻烦.....没有再traefik配置文件里面写对我个人来说方便。

那我现在怎么就用slb http  https代理方式了呢？

1. 当然了  首先是可以挂载多个证书了
1. 我在slb上面直接绑定了泛域名，后面的具体域名解析还是在我的traefik配置。但是我不用绑定证书了....
1. http  https的方式我可以把日志直接写入他的cos对象存储和腾讯云自己的日志服务（感觉也是一个kibana）可以直接分析日志啊.....

![image.png](https://img-blog.csdnimg.cn/img_convert/6bfbd82e0ccdf068eb7cb153e56167c2.png#align=left&display=inline&height=244&margin=[objectObject]&name=image.png&originHeight=489&originWidth=1676&size=41701&status=done&style=none&width=838)
综上所述，来实现一下我个人的过程与思路

1. 创建slb .slb绑定 work节点 80端口(这里我用的是负载均衡型，没有用传统型)，没有问题吧？老老实实ipv4了没有启用ipv6这个就看个人具体需求吧。

![image.png](https://img-blog.csdnimg.cn/img_convert/3b31852d3b61b3f67201c75c397f1a1f.png#align=left&display=inline&height=374&margin=[objectObject]&name=image.png&originHeight=748&originWidth=1275&size=65362&status=done&style=none&width=637.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/6f332352d5c769f347b398fa0ba50526.png#align=left&display=inline&height=278&margin=[objectObject]&name=image.png&originHeight=556&originWidth=687&size=26931&status=done&style=none&width=343.5)
使用了极度不要脸的方式 泛域名....因为我常用的也就这两个域名，具体的解析都还是我自己在traefik配置了。
![image.png](https://img-blog.csdnimg.cn/img_convert/f6fadafaaf63151bad81749038136293.png#align=left&display=inline&height=267&margin=[objectObject]&name=image.png&originHeight=534&originWidth=1450&size=34521&status=done&style=none&width=725)
关于证书 我这里可是扔好了 两个主二级域名，泛域名证书直接扔上了......
![image.png](https://img-blog.csdnimg.cn/img_convert/49422018f2f9b647fffc93c380925b53.png#align=left&display=inline&height=187&margin=[objectObject]&name=image.png&originHeight=374&originWidth=1115&size=27298&status=done&style=none&width=557.5)
四个后面配置我都绑定了80交给traefik处理吧。权重我都设置的一样的，有其他需求的可以根据自己需要设置呢。
![image.png](https://img-blog.csdnimg.cn/img_convert/b7ebbe0745cec6af1f2958b0b46e927e.png#align=left&display=inline&height=258&margin=[objectObject]&name=image.png&originHeight=516&originWidth=1376&size=42590&status=done&style=none&width=688)
## 2. 配置路由规则
  Traefik 应用已经部署完成，并且和slb负载均衡集成也大致完成了。但是想让外部访问 Kubernetes 内部服务，还需要配置路由规则，上面部署 Traefik 时开启了 `traefik dashboard`，这是 Traefik 提供的视图看板，所以，首先配置基于 `http` 的 `Traefik Dashboard` 路由规则，使外部能够访问 `Traefik Dashboard`。这里分别使用 `CRD`、`Ingress` 和 `Kubernetes Gateway API` 三种方式进行演示，过去版本常用的是CRD的方式。https的方式我就忽略了交给slb负载均衡层了。
### 1. CRD方式
过去我个人部署应用都是crd方式，自己老把这种方式叫做ingressroute方式。


```
cat <<EOF> traefik-dashboard-route-http.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
  namespaces: kube-system
spec:
  entryPoints:
  - web
  routes:
  - match: Host(\`traefik.saynaihe.com\`)
    kind: Rule
    services:
      - name: traefik
        port: 8080
EOF
kubectl apply -f traefik-dashboard-route-http.yaml       
```
![image.png](https://img-blog.csdnimg.cn/img_convert/f27ea6bbe0da83127e697ed607f90444.png#align=left&display=inline&height=143&margin=[objectObject]&name=image.png&originHeight=286&originWidth=770&size=23942&status=done&style=none&width=385)
关于   match: Host(\`traefik.saynaihe.com\`)  加转义符应该都能看明白了，不加转义符会是这样的
![image.png](https://img-blog.csdnimg.cn/img_convert/00bbe950a12cd3fc8316f24af0ba80b0.png#align=left&display=inline&height=284&margin=[objectObject]&name=image.png&originHeight=567&originWidth=876&size=50579&status=done&style=none&width=438)
我貌似又忘了加namespace 截图中，实际我可是加上了...老容易往事。哎，我不是两个泛域名吗 ？ 特意做了两个ingressoute 做下测试
![image.png](https://img-blog.csdnimg.cn/img_convert/3ee43493144ebb7ea5eaeeedf46dc043.png#align=left&display=inline&height=42&margin=[objectObject]&name=image.png&originHeight=83&originWidth=749&size=8609&status=done&style=none&width=374.5)
然后绑定本地hosts绑定host
```
C:\Windows\System32\drivers\etc
```
![image.png](https://img-blog.csdnimg.cn/img_convert/65953fe87d4996c00c229f6253570a80.png#align=left&display=inline&height=151&margin=[objectObject]&name=image.png&originHeight=301&originWidth=1235&size=52169&status=done&style=none&width=617.5)
遮挡的有点多....但是 这就是两个都路由过来了啊
![image.png](https://img-blog.csdnimg.cn/img_convert/ba6f31e52bb58c91b13126aa96237a89.png#align=left&display=inline&height=466&margin=[objectObject]&name=image.png&originHeight=932&originWidth=1621&size=194469&status=done&style=none&width=810.5)
关于https可以忽略了直接挂载在slb层了啊。然后http 强制跳转 https也可以在slb层上面配置了
![image.png](https://img-blog.csdnimg.cn/img_convert/bfa2a2fc8af78e3d0dc6019ccd3cb16d.png#align=left&display=inline&height=342&margin=[objectObject]&name=image.png&originHeight=683&originWidth=1523&size=111451&status=done&style=none&width=761.5)
流氓玩法强跳....测试也是成功的....
![image.png](https://img-blog.csdnimg.cn/img_convert/fe1db72829a2dc1602458e5d9069660f.png#align=left&display=inline&height=174&margin=[objectObject]&name=image.png&originHeight=348&originWidth=1561&size=25364&status=done&style=none&width=780.5)
### 2. Ingress方式
ingress的方式基本就是[https://kubernetes.io/zh/docs/concepts/services-networking/ingress/](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/) kubernetes 常见的ingress方式吧？
继续拿dashboard做演示
```
cat <<EOF> traefik-dashboard-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-dashboard-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik  
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: traefik1.saynaihe.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: traefik
            port:
              number: 8080
EOF
kubectl apply -f traefik-dashboard-ingress.yaml
```
![image.png](https://img-blog.csdnimg.cn/img_convert/52a4e18ada84a2664d8ef5be4e7688ee.png#align=left&display=inline&height=228&margin=[objectObject]&name=image.png&originHeight=456&originWidth=899&size=42553&status=done&style=none&width=449.5)
由于端口强制跳转了设置，直接https了哈哈哈验证完成
![image.png](https://img-blog.csdnimg.cn/img_convert/6cacdd9191ecfbb4298c2f129cf513c7.png#align=left&display=inline&height=444&margin=[objectObject]&name=image.png&originHeight=888&originWidth=1627&size=121141&status=done&style=none&width=813.5)
## 3、方式三：使用 Kubernetes Gateway API
关于`Kubernetes Gateway API` 可以通过`CRD` 方式创建路由规则
              CRD  自定义资源强调一下
    详情可以参考：[https://doc.traefik.io/traefik/v2.4/routing/providers/kubernetes-gateway/](https://doc.traefik.io/traefik/v2.4/routing/providers/kubernetes-gateway/)

- **GatewayClass：** GatewayClass 是基础结构提供程序定义的群集范围的资源。此资源表示可以实例化的网关类。一般该资源是用于支持多个基础设施提供商用途的，这里我们只部署一个即可。
- **Gateway：** Gateway 与基础设施配置的生命周期是 1:1。当用户创建网关时，GatewayClass 控制器会提供或配置一些负载平衡基础设施。
- **HTTPRoute：** HTTPRoute 是一种网关 API 类型，用于指定 HTTP 请求从网关侦听器到 API 对象（即服务）的路由行为。

![image.png](https://img-blog.csdnimg.cn/img_convert/784984193bf45635c12225480a9d7133.png#align=left&display=inline&height=423&margin=[objectObject]&name=image.png&originHeight=845&originWidth=1690&size=196554&status=done&style=none&width=845)
### 1. 创建 GatewayClass
****#**创建 GatewayClass 资源 kubernetes-gatewayclass.yaml 文件**
**参照： **[https://doc.traefik.io/traefik/v2.4/routing/providers/kubernetes-gateway/#kind-gatewayclass](https://doc.traefik.io/traefik/v2.4/routing/providers/kubernetes-gateway/#kind-gatewayclass)
```
cat <<EOF> kubernetes-gatewayclass.yaml
kind: GatewayClass
apiVersion: networking.x-k8s.io/v1alpha1
metadata:
  name: traefik
spec:
  # Controller is a domain/path string that indicates
  # the controller that is managing Gateways of this class.
  controller: traefik.io/gateway-controller
EOF
kubectl apply -f kubernetes-gatewayclass.yaml
```
### 2 配置 HTTP 路由规则 （Traefik Dashboard 为例）
**创建 Gateway 资源 http-gateway.yaml 文件**
```
cat <<EOF> http-gateway.yaml
apiVersion: networking.x-k8s.io/v1alpha1
kind: Gateway
metadata: 
  name: http-gateway
  namespace: kube-system
spec: 
  gatewayClassName: traefik
  listeners: 
    - protocol: HTTP
      port: 80
      routes: 
        kind: HTTPRoute
        namespaces:
          from: All
        selector:
          matchLabels:
            app: traefik
EOF
kubectl apply -f http-gateway.yaml            
```
**创建 HTTPRoute 资源 traefik-httproute.yaml 文件**
```
cat <<EOF> traefik-httproute.yaml
apiVersion: networking.x-k8s.io/v1alpha1
kind: HTTPRoute
metadata:
  name: traefik-dashboard-httproute
  namespace: kube-system
  labels:
    app: traefik
spec:
  hostnames:
    - "traefi2.saynaihe.com"
  rules:
    - matches:
        - path:
            type: Prefix
            value: /
      forwardTo:
        - serviceName: traefik
          port: 8080
          weight: 1
EOF
kubectl apply -f traefik-httproute.yaml            
```
这里就出问题了.....，无法访问，仔细看了下文档[https://doc.traefik.io/traefik/providers/kubernetes-gateway/](https://doc.traefik.io/traefik/providers/kubernetes-gateway/)
![image.png](https://img-blog.csdnimg.cn/img_convert/b6c18a95682a987fa06bcf07974404d5.png#align=left&display=inline&height=359&margin=[objectObject]&name=image.png&originHeight=718&originWidth=1588&size=124387&status=done&style=none&width=794)
全部删除一次重新部署吧将2.5中版本变成v0.1.0就好了....图就不上了基本步骤是一样的。
![image.png](https://img-blog.csdnimg.cn/img_convert/04920fa7cf9f829c79d4c9224d390dc5.png#align=left&display=inline&height=412&margin=[objectObject]&name=image.png&originHeight=823&originWidth=1570&size=104263&status=done&style=none&width=785)
注： 都没有做域名解析，本地绑定了host。 saynaihe.com域名只是做演示。没有实际搞....因为我没有做备案。现在不备案的基本绑上就被扫描到封了。用正式域名做的试验。另外养成的习惯用CRD习惯了...部署应用基本个人都用了CRD的方式  ---ingressroute。ingress的方式是更适合从ingress-nginx迁移到traefik使用了。至于Kubernetes Gateway API个人还是图个新鲜，没有整明白。v0.2.0不能用...就演示下了.


