---
layout: post
title: Create you course K8s cluster-创建课程所需的k8s集群
date: 2021-02-01 10:00:00
category: cks
tags:  kubernetes cks
author: duiniwukenaihe
---
* content
{:toc

注： 由于限制不能谷歌云绑定银联卡了，直接拿两台腾讯云服务器做课程实例
线上跑的是自建的集群搭建方式详见：[https://duiniwukenaihe.github.io/2020/07/22/tencent-slb-kubeadm-ha/](https://duiniwukenaihe.github.io/2020/07/22/tencent-slb-kubeadm-ha/)（跑了两个集群，其实还是跑的1.16版本，只进行了小版本升级现为1.16.15版本）
关于安全组配置就不详细说明了，由于是个人测试这里也没有做安全组策略，直接开放了ALL，ssh端口也没有做更改，当然了密码设置还是符合个人的安全策略的。由于测试环境不做各种系统优化，复杂配置了。直接就按照课程的操作来了。
 10.0.2.6  cks-master
 10.0.2.17 cks-work
更改主机名 hostnamectl set-hostname cks-xxx
配置如下：
![image.png](https://img-blog.csdnimg.cn/img_convert/5ce6bbb0af354b3e0f82b9bdc7890ad9.png#align=left&display=inline&height=705&margin=[objectObject]&name=image.png&originHeight=705&originWidth=696&size=40675&status=done&style=none&width=696)
(由于kill的课程是在国外的，apt仓库都是直接用的国外的，镜像仓库直接用的google的，切github仓库进行了版本更新故，修改了脚本)：
## 1. 10.0.2.6  cks-master 节点操作步骤：
### 1.1  master节点初始化
sh    install_master.sh
```bash
#!/bin/sh
# Source: http://kubernetes.io/docs/getting-started-guides/kubeadm/
### setup terminal
apt-get install -y bash-completion binutils
echo 'colorscheme ron' >> ~/.vimrc
echo 'set tabstop=2' >> ~/.vimrc
echo 'set shiftwidth=2' >> ~/.vimrc
echo 'set expandtab' >> ~/.vimrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'alias c=clear' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
sed -i '1s/^/force_color_prompt=yes\n/' ~/.bashrc
### install k8s and docker
apt-get remove -y docker.io kubelet kubeadm kubectl kubernetes-cni
apt-get autoremove -y
apt-get install -y etcd-client vim build-essential
systemctl daemon-reload
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
KUBE_VERSION=1.19.3
apt-get update
apt-get install -y docker.io kubelet=${KUBE_VERSION}-00 kubeadm=${KUBE_VERSION}-00 kubectl=${KUBE_VERSION}-00 kubernetes-cni=0.8.7-00
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
# Restart docker.
systemctl daemon-reload
systemctl restart docker
# start docker on reboot
systemctl enable docker
docker info | grep -i "storage"
docker info | grep -i "cgroup"
systemctl enable kubelet && systemctl start kubelet
### init k8s
rm /root/.kube/config
kubeadm reset -f
kubeadm init --kubernetes-version=${KUBE_VERSION} --ignore-preflight-errors=NumCPU --skip-token-print
mkdir -p ~/.kube
sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
echo
echo "### COMMAND TO ADD A WORKER NODE ###"
kubeadm token create --print-join-command --ttl 0
```
![image.png](https://img-blog.csdnimg.cn/img_convert/3472aa5e16e8233d20bfc0857bf0b1fb.png#align=left&display=inline&height=544&margin=[objectObject]&name=image.png&originHeight=544&originWidth=1301&size=60961&status=done&style=none&width=1301)
### 1.2.  下载所需要镜像
kubeadm config images list --kubernetes-version 1.19.3 确定1.19.3版本所需要的镜像版本，在阿里云镜像仓库下载并且修改镜像标签为k8s.gcr.io镜像仓库标签，当然了也可以采用创建kubeadm初始化文件的方式修改镜像仓库为阿里云或者其他国内镜像仓库。至于不同版本之间都是大同小异。
![image.png](https://img-blog.csdnimg.cn/img_convert/d06b78ace9aa285b0f958d9e1b326fb7.png#align=left&display=inline&height=510&margin=[objectObject]&name=image.png&originHeight=510&originWidth=1280&size=50913&status=done&style=none&width=1280)


   sh images.sh
```bash
#!/bin/bash
images=(
    kube-apiserver:v1.19.3
    kube-controller-manager:v1.19.3
    kube-scheduler:v1.19.3
    kube-proxy:v1.19.3
    pause:3.2
    etcd:3.4.13-0
    coredns:1.7.0
)

for imageName in ${images[@]};do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
  docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done

```

![image.png](https://img-blog.csdnimg.cn/img_convert/4d0e1d52b71356f50d7e3f0213205af2.png#align=left&display=inline&height=717&margin=[objectObject]&name=image.png&originHeight=717&originWidth=1378&size=160215&status=done&style=none&width=1378)
注：碰到的好玩的注意的：

1. copy有格式的内容到linux如何保持原有的格式？ vim  :set parste
1. ubuntu执行bash  显示：images.sh: 2: images.sh: Syntax error: "(" unexpected  why? 详见：[https://blog.csdn.net/u014470581/article/details/51493150/](https://blog.csdn.net/u014470581/article/details/51493150/)

          sudo dpkg-reconfigure dash
![image.png](https://img-blog.csdnimg.cn/img_convert/5eb2a9be5e0d238223f7d8fbe79049ee.png#align=left&display=inline&height=648&margin=[objectObject]&name=image.png&originHeight=648&originWidth=1286&size=32265&status=done&style=none&width=1286)
           选择no 保存 就ok了。

3. 关于下载镜像。下载镜像是dokcer去下载的自己把控执行下载镜像脚本的时间了，当install_master.sh脚本安装完docker的过程中就可以下载镜像了。当然了 也可以安装自己的节奏来了，不一定用他教程上面的了，按照他的步骤就是纯属为了加深下课程的理解。
3. 当然了还有你想自己修改的，比如网络插件，集群节点的网络规划网段，都可以安装自己想的修改了。



## 2. 10.0.2.17 cks-work 节点操作步骤：
### 2.1. work节点执行初始化脚本
注： 与master脚本修改大同小异
           sh install_work.sh
```bash

# Source: http://kubernetes.io/docs/getting-started-guides/kubeadm/

### setup terminal
apt-get install -y bash-completion binutils
echo 'colorscheme ron' >> ~/.vimrc
echo 'set tabstop=2' >> ~/.vimrc
echo 'set shiftwidth=2' >> ~/.vimrc
echo 'set expandtab' >> ~/.vimrc
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'alias c=clear' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
sed -i '1s/^/force_color_prompt=yes\n/' ~/.bashrc


### install k8s and docker
apt-get remove -y docker.io kubelet kubeadm kubectl kubernetes-cni
apt-get autoremove -y
apt-get install -y etcd-client vim build-essential

systemctl daemon-reload
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
KUBE_VERSION=1.19.3
apt-get update
apt-get install -y docker.io kubelet=${KUBE_VERSION}-00 kubeadm=${KUBE_VERSION}-00 kubectl=${KUBE_VERSION}-00 kubernetes-cni=0.8.7-00

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "storage-driver": "overlay2"
}
EOF
mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker

# start docker on reboot
systemctl enable docker

docker info | grep -i "storage"
docker info | grep -i "cgroup"

systemctl enable kubelet && systemctl start kubelet


### init k8s
kubeadm reset -f
systemctl daemon-reload
service kubelet start

echo
echo "EXECUTE ON MASTER: kubeadm token create --print-join-command --ttl 0"
echo "THEN RUN THE OUTPUT AS COMMAND HERE TO ADD AS WORKER"
echo

```
### 2.2.  下载镜像
 sh images.sh
```bash
#!/bin/bash
images=(
    kube-apiserver:v1.19.3
    kube-controller-manager:v1.19.3
    kube-scheduler:v1.19.3
    kube-proxy:v1.19.3
    pause:3.2
    etcd:3.4.13-0
    coredns:1.7.0
)

for imageName in ${images[@]};do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
  docker tag  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi  registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```
### 2.3. work节点加入集群


```bash

root@VM-2-17-ubuntu:~# kubeadm join 10.0.2.6:6443 --token ux1gld.q5bzt4aq6p87fnuv     --discovery-token-ca-cert-hash sha256:b9638833e81b1e8042ea10ec2a958d08196ffd31f0f6a5b81f40e526f7d12944 
```
![image.png](https://img-blog.csdnimg.cn/img_convert/6ee7c08825534c8bc4c382504e6de609.png#align=left&display=inline&height=718&margin=[objectObject]&name=image.png&originHeight=718&originWidth=1333&size=103567&status=done&style=none&width=1333)


## 3. 验证集群安装


over  然后master节点
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```bash
root@cks-master:~# kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
cks-master       Ready      master   14m   v1.19.3
vm-2-17-ubuntu   NotReady   <none>   43s   v1.19.3
root@cks-master:~# kubectl get nodes
NAME             STATUS   ROLES    AGE   VERSION
cks-master       Ready    master   14m   v1.19.3
vm-2-17-ubuntu   Ready    <none>   51s   v1.19.3
root@cks-master:~# kubectl get pods -n kube-system
NAME                                 READY   STATUS            RESTARTS   AGE
coredns-f9fd979d6-qj2jt              1/1     Running           0          14m
coredns-f9fd979d6-xwl4k              1/1     Running           0          14m
etcd-cks-master                      1/1     Running           0          14m
kube-apiserver-cks-master            1/1     Running           0          14m
kube-controller-manager-cks-master   1/1     Running           0          14m
kube-proxy-9x4pr                     1/1     Running           0          68s
kube-proxy-kf7ns                     1/1     Running           0          14m
kube-scheduler-cks-master            1/1     Running           0          14m
weave-net-2b29f                      0/2     PodInitializing   0          68s
weave-net-j78m4                      2/2     Running           1          14m

```
 
嗯？强迫症犯了 突然发现work节点忘记了修改主机名......。操作步骤应该是：

1. cks-master节点 驱逐删除vm-2-17-ubuntu节点（kubectl delete node cks-work，由于是新的节点就不跑驱逐和设置不可调度了）
1. vm-2-17-ubuntu节点操作
```bash
1.  更改主机名 hostnamectl  set-hostname cks-work
2.  初始化kubeadm   kubeadm reset
3.  重新加入master节点    kubeadm join 10.0.2.6:6443 --token ux1gld.q5bzt4aq6p87fnuv     --discovery-token-ca-cert-hash sha256:b9638833e81b1e8042ea10ec2a958d08196ffd31f0f6a5b81f40e526f7d12944
```
![image.png](https://img-blog.csdnimg.cn/img_convert/2d26917b8abe090f5764c0e41fde238c.png#align=left&display=inline&height=664&margin=[objectObject]&name=image.png&originHeight=664&originWidth=1050&size=86525&status=done&style=none&width=1050)
![image.png](https://img-blog.csdnimg.cn/img_convert/01d23dd15dbe004a8ebe2c6ee2153f96.png#align=left&display=inline&height=634&margin=[objectObject]&name=image.png&originHeight=634&originWidth=1320&size=95890&status=done&style=none&width=1320)
ok最终cks-master节点操作kubectl get node如下 :
![image.png](https://img-blog.csdnimg.cn/img_convert/44e294a9982d7a84dc762c7e87572c1b.png#align=left&display=inline&height=539&margin=[objectObject]&name=image.png&originHeight=539&originWidth=635&size=60619&status=done&style=none&width=635)
       
         
 







