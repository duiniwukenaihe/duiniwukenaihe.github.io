---
layout: post
title: Prometheus Operator 监控 Traefik V2.4
date: 2021-04-06 01:00:00
category: kubernetes1.20
tags:  kubernetes Prometheus traefik
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
traefik搭建方式如下：
[https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7](https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7) 。
Prometheus-oprator搭建方式如下：
[https://www.yuque.com/duiniwukenaihe/ehb02i/tm6vl7](https://www.yuque.com/duiniwukenaihe/ehb02i/tm6vl7)。
Prometheus的文档写了grafana添加了traefik的监控模板。但是现在仔细一看。traefik的监控图是空的，Prometheus的 target也没有对应traefik的监控。
现在配置下添加traefik服务发现以及验证一下grafana的图表。
# 1. Prometheus Operator 监控 Traefik V2.4
## 1.1.  [Traefik 配置文件设置 Prometheus](http://www.mydlq.club/article/19/#%E4%B8%80traefik-%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E8%AE%BE%E7%BD%AE-prometheus)
参照[https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7](https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7)。配置中默认开启了默认的Prometheus监控。
![image.png](https://img-blog.csdnimg.cn/img_convert/8aa43201a68a8455a88a9768e7df619b.png#align=left&display=inline&height=348&margin=[objectObject]&name=image.png&originHeight=696&originWidth=1512&size=163570&status=done&style=none&width=756)
[https://doc.traefik.io/traefik/observability/metrics/prometheus/](https://doc.traefik.io/traefik/observability/metrics/prometheus/)可参照traefik官方文档。
## 1.2、Traefik Service 设置标签
### 1.2.1 查看traefik service
```
$  kubectl get svc -n kube-system
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
cilium-agent                  ClusterIP   None             <none>        9095/TCP                       15d
etcd-k8s                      ClusterIP   None             <none>        2379/TCP                       8d
hubble-metrics                ClusterIP   None             <none>        9091/TCP                       15d
hubble-relay                  ClusterIP   172.254.38.50    <none>        80/TCP                         15d
hubble-ui                     ClusterIP   172.254.11.239   <none>        80/TCP                         15d
kube-controller-manager       ClusterIP   None             <none>        10257/TCP                      8d
kube-controller-manager-svc   ClusterIP   None             <none>        10252/TCP                      6d
kube-dns                      ClusterIP   172.254.0.10     <none>        53/UDP,53/TCP,9153/TCP         15d
kube-scheduler                ClusterIP   None             <none>        10259/TCP                      8d
kube-scheduler-svc            ClusterIP   None             <none>        10251/TCP                      6d
kubelet                       ClusterIP   None             <none>        10250/TCP,10255/TCP,4194/TCP   8d
traefik                       ClusterIP   172.254.12.88    <none>        80/TCP,443/TCP,8080/TCP        11d

```
### 1.2.2、编辑该 Service 设置 Label
```
 kubectl edit service traefik -n kube-system
```
设置 Label “app: traefik”
![image.png](https://img-blog.csdnimg.cn/img_convert/56c31e5a392ea6175d0e67c296bbbaed.png#align=left&display=inline&height=383&margin=[objectObject]&name=image.png&originHeight=766&originWidth=1710&size=76502&status=done&style=none&width=855)
参照的是traefik-deploy.yaml 中的app:traefik这个标签用了，当然了也可以自己定义下用下别的......
![image.png](https://img-blog.csdnimg.cn/img_convert/0a6df55ace6714b94049cc808e179afc.png#align=left&display=inline&height=489&margin=[objectObject]&name=image.png&originHeight=978&originWidth=1386&size=134603&status=done&style=none&width=693)
## 1.3、Prometheus Operator 配置监控规则


**traefik-service-monitoring.yaml**
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name:  traefik
  namespace: monitoring
  labels:
     app: traefik
spec:
  jobLabel: traefik-metrics
  selector:
    matchLabels:
      app: traefik
  namespaceSelector:
    matchNames:
    - kube-system
  endpoints:
  - port: admin
    path: /metrics

```
**创建该Service Monitor**
```
$ kubectl apply -f traefik-monitor.yaml
```


## 1.4、查看 Prometheus  target规则是否生效
![image.png](https://img-blog.csdnimg.cn/img_convert/e1ae10688f192c26f4693080d7a00982.png#align=left&display=inline&height=502&margin=[objectObject]&name=image.png&originHeight=1004&originWidth=1693&size=164883&status=done&style=none&width=846.5)
## 1.5 查看grafana中的traefik仪表盘是否有数据生成图表
![image.png](https://img-blog.csdnimg.cn/img_convert/ea0defda356db31a842670767710e04c.png#align=left&display=inline&height=496&margin=[objectObject]&name=image.png&originHeight=993&originWidth=1889&size=194605&status=done&style=none&width=944.5)
嗯 这也算是添加target的一个例子。下次写下配置下自动发现规则？


