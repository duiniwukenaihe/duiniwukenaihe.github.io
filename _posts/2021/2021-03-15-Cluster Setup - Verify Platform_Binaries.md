---
layout: post
title: Cluster Setup - Verify Platform_Binaries
date: 2021-03-15 10:00:00
category: cks
tags:  kubernetes cks Verify Platform_Binaries
author: duiniwukenaihe
---
* content
{:toc


## 释意：
## Verify platform binaries  验证平台的二进制文件


# 1. Hashes  哈希散列
详见知乎：[https://zhuanlan.zhihu.com/p/37165658](https://zhuanlan.zhihu.com/p/37165658)关于哈希算法与MD5、SHA讲的很是详细。还有csdn的[https://blog.csdn.net/ljy1988123/article/details/51506578](https://blog.csdn.net/ljy1988123/article/details/51506578)
## 1.1 Theory  and Hashes-理论与哈希
哈希算法有两个评价标准，一个是无法回源，一个是随机性（碰撞概率小），一个是计算速度。常见的算法 Hash  SHA  MD5
![image.png](https://img-blog.csdnimg.cn/img_convert/ff5fc4405cd55a4cd4b93a229d5bea8c.png#align=left&display=inline&height=483&margin=[objectObject]&name=image.png&originHeight=483&originWidth=865&size=68716&status=done&style=none&width=865)
## 1.2 Download and verify binaries  下载并验证二进制文件



### 1.2.1 确认版本
![image.png](https://img-blog.csdnimg.cn/img_convert/3e2426dcf65c9cd0692b106df4288d16.png#align=left&display=inline&height=61&margin=[objectObject]&name=image.png&originHeight=122&originWidth=977&size=13065&status=done&style=none&width=488.5)
### 1.2.2 Dowload kubernetes release from github
![image.png](https://img-blog.csdnimg.cn/img_convert/bd407e3f634974e65186c2bd2451ee7a.png#align=left&display=inline&height=428&margin=[objectObject]&name=image.png&originHeight=855&originWidth=1320&size=71375&status=done&style=none&width=660)
![image.png](https://img-blog.csdnimg.cn/img_convert/4ee39de16db7924737483c759d7a54b8.png#align=left&display=inline&height=454&margin=[objectObject]&name=image.png&originHeight=908&originWidth=1761&size=114003&status=done&style=none&width=880.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/96e4286f08cab0bf39e3ff6b5d51c1af.png#align=left&display=inline&height=504&margin=[objectObject]&name=image.png&originHeight=1008&originWidth=1781&size=152138&status=done&style=none&width=890.5)
### 1.2.3 verify downloaded files  验证下载文件
```html
sha512sum   kubernetes-server-linux-amd64.tar.gz >compare
```
![image.png](https://img-blog.csdnimg.cn/img_convert/7b6c2b9d47ba9fb2ff2e7c5695c94369.png#align=left&display=inline&height=50&margin=[objectObject]&name=image.png&originHeight=100&originWidth=1408&size=18541&status=done&style=none&width=704)
![image.png](https://img-blog.csdnimg.cn/img_convert/c601d2062d06d6b74b02a13b59ee1235.png#align=left&display=inline&height=366&margin=[objectObject]&name=image.png&originHeight=732&originWidth=1917&size=115652&status=done&style=none&width=958.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/d394f26502896210436e397185322181.png#align=left&display=inline&height=88&margin=[objectObject]&name=image.png&originHeight=176&originWidth=1307&size=37413&status=done&style=none&width=653.5)

将github页面的sha512 hash 跟本地验证的对比 oK一致
## 1.3. Verify binaries from container验证容器中的二进制文件


![image.png](https://img-blog.csdnimg.cn/img_convert/1cbe4d7e8382497d0af0d3866be58410.png#align=left&display=inline&height=241&margin=[objectObject]&name=image.png&originHeight=483&originWidth=868&size=111551&status=done&style=none&width=434)
### 1.3.1 解压 从github下载的1.9.3的kubernetes压缩包，以kube-apiserver为例，验证解压文件夹内的kuber-apiserver的 sha512sum.并将其写入compare文件
```html
root@cks-master:~/hash# tar -zxf kubernetes-server-linux-amd64.tar.gz 
root@cks-master:~/hash# ls kubernetes
addons  kubernetes-src.tar.gz  LICENSES  server
root@cks-master:~/hash# ls kubernetes/server/bin/
apiextensions-apiserver  kube-aggregator  kube-apiserver.docker_tag  kube-controller-manager             kube-controller-manager.tar  kubelet     kube-proxy.docker_tag  kube-scheduler             kube-scheduler.tar
kubeadm                  kube-apiserver   kube-apiserver.tar         kube-controller-manager.docker_tag  kubectl                      kube-proxy  kube-proxy.tar         kube-scheduler.docker_tag  mounter
root@cks-master:~/hash# sha512sum kubernetes/server/bin/kube-apiserver
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  kubernetes/server/bin/kube-apiserver
root@cks-master:~/hash# sha256sum kubernetes/server/bin/kube-apiserver > compare

```
### 1.3.2 再次确认下 kubernetes集群中kube-apiserver运行的是同版本1.19.3，然后container中是没有bash  sh的。使用docker cp copy到container-fs文件夹，当然其实也可以用kubectl  cp查找文件夹中kube-apiserver文件。然后sh512sum  ，追加如compare文件。对哈希值进行对比：
```html
root@cks-master:~/hash# kubectl get pods -n kube-system|grep api
kube-apiserver-cks-master            1/1     Running   0          42d
root@cks-master:~/hash#  kubectl get pod kube-apiserver-cks-master -n kube-system -o yaml|grep image
            f:image: {}
            f:imagePullPolicy: {}
    image: k8s.gcr.io/kube-apiserver:v1.19.3
    imagePullPolicy: IfNotPresent
    image: k8s.gcr.io/kube-apiserver:v1.19.3
    imageID: docker://sha256:a301be0cd44bb11162da49b9c55fc5d137f493bdefcf80226378204be403fa41
root@cks-master:~/hash# kubectl exec -it kube-apiserver-cks-master bash -n kube-system
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
OCI runtime exec failed: exec failed: container_linux.go:349: starting container process caused "exec: \"bash\": executable file not found in $PATH": unknown
command terminated with exit code 126
root@cks-master:~/hash# docker ps |grep apiserver
72c54882e5c0        a301be0cd44b           "kube-apiserver --ad…"   6 weeks ago         Up 6 weeks                              k8s_kube-apiserver_kube-apiserver-cks-master_kube-system_a2aef6235c950d78a8c2a8f52536f35e_0
4045b57cf208        k8s.gcr.io/pause:3.2   "/pause"                 6 weeks ago         Up 6 weeks                              k8s_POD_kube-apiserver-cks-master_kube-system_a2aef6235c950d78a8c2a8f52536f35e_0
root@cks-master:~/hash# docker cp 72c54882e5c0:/ container-fs
root@cks-master:~/hash# find container-fs/ -name kube-apiserver
container-fs/usr/local/bin/kube-apiserver
root@cks-master:~/hash# sha512sum container-fs/usr/local/bin/kube-apiserver
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  container-fs/usr/local/bin/kube-apiserver
root@cks-master:~/hash# sha512sum container-fs/usr/local/bin/kube-apiserver >> co
compare       container-fs/ 
root@cks-master:~/hash# sha512sum container-fs/usr/local/bin/kube-apiserver >> compare 
root@cks-master:~/hash# cat compare 
3bda7b83d70fc762759f88a93b760355a6c1023be959d613a3faf113b975200c  kubernetes/server/bin/kube-apiserver
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  container-fs/usr/local/bin/kube-apiserver
root@cks-master:~/hash# rm -rf compare 
root@cks-master:~/hash# sha512sum kubernetes/server/bin/kube-apiserver
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  kubernetes/server/bin/kube-apiserver
root@cks-master:~/hash# sha512sum kubernetes/server/bin/kube-apiserver > compare
root@cks-master:~/hash# cat compare 
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  kubernetes/server/bin/kube-apiserver
root@cks-master:~/hash# sha512sum container-fs/usr/local/bin/kube-apiserver >> compare
root@cks-master:~/hash# cat compare 
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  kubernetes/server/bin/kube-apiserver
49b3a12ee597ea3bf9ece98accb62018ef758e4766ae44e24386838306cf69a2bc5dc7f8c0b728abecb972a4b651271f140bfdf0047e483c1556662cbd5b914a  container-fs/usr/local/bin/kube-apiserver

```


![image.png](https://img-blog.csdnimg.cn/img_convert/519a8bac91d541caa683965dd55e0189.png#align=left&display=inline&height=245&margin=[objectObject]&name=image.png&originHeight=490&originWidth=1724&size=120833&status=done&style=none&width=862)
验证通过 OK


