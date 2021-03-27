---
layout: post
title: centos8+kubeadm1.20.5+cilium+hubble环境搭建
date: 2021-03-25 17:00:00
category: kubernetes1.20
tags:  kubernetes cilium hubble
author: duiniwukenaihe
---
* content
{:toc}
# 前言
腾讯云绑定用户，开始使用过腾讯云的tke1.10版本。鉴于各种原因选择了自建。线上kubeadm自建kubernetes集群1.16版本（小版本升级到1.16.15）。kubeadm+haproxy+slb+flannel搭建高可用集群，集群启用ipvs。对外服务使用slb绑定traefik tcp 80  443端口对外映射（这是历史遗留问题，过去腾讯云slb不支持挂载多证书，这样也造成了无法使用slb的日志投递功能，现在slb已经支持了多证书的挂载，可以直接使用http http方式了）。生产环境当时搭建仓库没有使用腾讯云的块存储，直接使用cbs。直接用了local disk,还有nfs的共享存储。前几天整了个项目的压力测试，然后使用nfs存储的项目IO直接就飙升了。生产环境不建议使用。准备安装kubernetes 1.20版本，并使用cilium组网。hubble替代kube-proxy 体验一下ebpf。另外也直接上containerd。dockershim的方式确实也浪费资源的。这样也是可以减少资源开销，部署速度的。反正就是体验一下各种最新功能：
![image.png](https://img-blog.csdnimg.cn/img_convert/7b1e4478c35beb59c94356c0fe860813.png#align=left&display=inline&height=404&margin=[objectObject]&name=image.png&originHeight=808&originWidth=816&size=214004&status=done&style=none&width=408)
![image.png](https://img-blog.csdnimg.cn/img_convert/8501dd6e6766bdbf93a7b960627c0ba7.png#align=left&display=inline&height=168&margin=[objectObject]&name=image.png&originHeight=336&originWidth=782&size=57458&status=done&style=none&width=391)
图片引用自：[https://blog.kelu.org/tech/2020/10/09/the-diff-between-docker-containerd-runc-docker-shim.html](https://blog.kelu.org/tech/2020/10/09/the-diff-between-docker-containerd-runc-docker-shim.html)
# 环境准备：
注：master节点4核心8G配置。work节点16核32G。腾讯云S5云主机
| 主机名 | ip | 系统 | 内核 |
| --- | --- | --- | --- |
| sh-master-01 | 10.3.2.5  | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-master-02 | 10.3.2.13 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-master-03 | 10.3.2.16 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-01 | 10.3.2.2 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-02 | 10.3.2.3 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-03 | 10.3.2.4 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |

注： 用centos8是为了懒升级内核版本了。centos7内核版本3.10确实有些老了。但是同样的centos8 kubernetes源是没有的，只能使用centos7的源。
VIP  slb地址：10.3.2.12（因为内网没有使用域名的需求，直接用了传统型内网负载，为了让slb映射端口与本地端口一样中间加了一层haproxy代理本地6443.然后slb代理8443端口为6443.）。
![image.png](https://img-blog.csdnimg.cn/img_convert/bb980270732f984faf857bec80061fd3.png#align=left&display=inline&height=236&margin=[objectObject]&name=image.png&originHeight=472&originWidth=550&size=22601&status=done&style=none&width=275)
![image.png](https://img-blog.csdnimg.cn/img_convert/2a8c7c5c02c6326fb072b2b80ce7cb7f.png#align=left&display=inline&height=416&margin=[objectObject]&name=image.png&originHeight=831&originWidth=1210&size=55052&status=done&style=none&width=605)
# 1. 系统初始化：
注：由于环境是部署在公有云的，使用了懒人方法。直接初始化了一台server.然后其他的直接都是复制的方式搭建的。
## 1. 更改主机名

```html
hostnamectl set-hostname sh-master-01
cat /etc/hosts
```
![image.png](https://img-blog.csdnimg.cn/img_convert/9c6027a9982a8d069d6e323cd5f70a46.png#align=left&display=inline&height=81&margin=[objectObject]&name=image.png&originHeight=161&originWidth=1118&size=16245&status=done&style=none&width=559)
就是举个例子了。我的host文件只在三台master节点写了，work节点都没有写的.......


## 2. 关闭swap交换分区
```
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
```


## 3. 关闭selinux
```
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config
```


## 4. 关闭防火墙
```html
systemctl disable --now firewalld
chkconfig firewalld off
```
## 5. 调整文件打开数等配置
```html
cat> /etc/security/limits.conf <<EOF
* soft nproc 1000000
* hard nproc 1000000
* soft nofile 1000000
* hard nofile 1000000
* soft  memlock  unlimited
* hard memlock  unlimited
EOF
```
当然了这里最好的其实是/etc/security/limits.d目录下生成一个新的配置文件。避免修改原来的总配置文件、这也是推荐使用的方式。
## 6. yum  update 八仙过海各显神通吧，安装自己所需的习惯的应用
```html
yum update
yum -y install  gcc bc gcc-c++ ncurses ncurses-devel cmake elfutils-libelf-devel openssl-devel flex* bison* autoconf automake zlib* fiex* libxml* ncurses-devel libmcrypt* libtool-ltdl-devel* make cmake  pcre pcre-devel openssl openssl-devel   jemalloc-devel tlc libtool vim unzip wget lrzsz bash-comp* ipvsadm ipset jq sysstat conntrack libseccomp conntrack-tools socat curl wget git conntrack-tools psmisc nfs-utils tree bash-completion conntrack libseccomp net-tools crontabs sysstat iftop nload strace bind-utils tcpdump htop telnet lsof
```


## 7. ipvs添加（centos8内核默认4.18.内核4.19不包括4.19的是用这个）
```html
:> /etc/modules-load.d/ipvs.conf
module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
br_netfilter
  )
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
```


内核大于等于4.19的
```html
:> /etc/modules-load.d/ipvs.conf
module=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
br_netfilter
  )
for kernel_module in ${module[@]};do
    /sbin/modinfo -F filename $kernel_module |& grep -qv ERROR && echo $kernel_module >> /etc/modules-load.d/ipvs.conf || :
done
```
这个地方我想我开不开ipvs应该没有多大关系了吧？ 因为我网络组件用的cilium  hubble。网络用的是ebpf。没有用iptables  ipvs吧？至于配置ipvs算是原来部署养成的习惯
加载ipvs模块
```html
systemctl daemon-reload
systemctl enable --now systemd-modules-load.service
```


查询ipvs是否加载
```html
#  lsmod | grep ip_vs
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 172032  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          172032  6 xt_conntrack,nf_nat,xt_state,ipt_MASQUERADE,xt_CT,ip_vs
nf_defrag_ipv6         20480  4 nf_conntrack,xt_socket,xt_TPROXY,ip_vs
libcrc32c              16384  3 nf_conntrack,nf_nat,ip_vs
```


## 8. 优化系统参数(不一定是最优，各取所有)
```html
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
# 要求iptables不对bridge的数据进行处理
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
vm.overcommit_memory=1
vm.panic_on_oom=0
vm.swappiness = 0
EOF

sysctl --system
```


## 9. containerd安装
  dnf  与yum  centos8的变化，具体的自己去看了呢。差不多吧.......
```html
dnf install dnf-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum update -y && sudo yum install -y containerd.io
containerd config default > /etc/containerd/config.toml
# 替换 containerd 默认的 sand_box 镜像，编辑 /etc/containerd/config.toml

sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"

# 重启containerd
$ systemctl daemon-reload
$ systemctl restart containerd

```
其他的配置一个是启用SystemdCgroup另外一个是添加了本地镜像库，账号密码（直接使用了腾讯云的仓库）。
![image.png](https://img-blog.csdnimg.cn/img_convert/0a5572f643ca79030fb0fb5cf7fa8bff.png#align=left&display=inline&height=241&margin=[objectObject]&name=image.png&originHeight=482&originWidth=941&size=56294&status=done&style=none&width=470.5)
## 10. 配置 CRI 客户端 crictl
```html
cat <<EOF > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# 验证是否可用（可以顺便验证一下私有仓库）
crictl  pull nginx:alpine
crictl  rmi  nginx:alpine
crictl  images
```


## 11. 安装 Kubeadm(centos8没有对应yum源使用centos7的阿里云yum源)
```html
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
# 删除旧版本，如果安装了
yum remove kubeadm kubectl kubelet kubernetes-cni cri-tools socat 
# 查看所有可安装版本 下面两个都可以啊
# yum list --showduplicates kubeadm --disableexcludes=kubernetes
# 安装指定版本用下面的命令
# yum -y install kubeadm-1.20.5 kubectl-1.20.5 kubelet-1.20.5
or 
# yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# 默认安装最新稳定版，当前版本1.20.5
yum install kubeadm

# 开机自启
systemctl enable kubelet.service

```
## 12. 修改kubelet配置
```html
vi /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS= --cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock
```
## 13 . journal 日志相关避免日志重复搜集，浪费系统资源。修改systemctl启动的最小文件打开数量,关闭ssh反向dns解析.设置清理日志，最大200m(可根据个人需求设置)
```
sed -ri 's/^\$ModLoad imjournal/#&/' /etc/rsyslog.conf
sed -ri 's/^\$IMJournalStateFile/#&/' /etc/rsyslog.conf
sed -ri 's/^#(DefaultLimitCORE)=/\1=100000/' /etc/systemd/system.conf
sed -ri 's/^#(DefaultLimitNOFILE)=/\1=100000/' /etc/systemd/system.conf
sed -ri 's/^#(UseDNS )yes/\1no/' /etc/ssh/sshd_config
journalctl --vacuum-size=200M
```


# 2. master节点操作
## 1 . 安装haproxy
```html
yum install haproxy
```
```html
cat <<EOF >  /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend kubernetes
    bind *:8443              #配置端口为8443
    mode tcp
    default_backend kubernetes
#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend kubernetes           #后端服务器，也就是说访问10.3.2.12:6443会将请求转发到后端的三台，这样就实现了负载均衡
    balance roundrobin               
    server master1  10.3.2.5:6443 check maxconn 2000
    server master2  10.3.2.13:6443 check maxconn 2000
    server master3  10.3.2.16:6443 check maxconn 2000
EOF
 systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy
```


嗯 slb绑定端口
![image.png](https://img-blog.csdnimg.cn/img_convert/e530a86097215a4e70c71a52de871eea.png#align=left&display=inline&height=347&margin=[objectObject]&name=image.png&originHeight=693&originWidth=1473&size=53528&status=done&style=none&width=736.5)
## 2. sh-master-01节点初始化
### 1.生成config配置文件
```html
kubeadm config print init-defaults > config.yaml
```
下面的图就是举个例子.......
![image.png](https://img-blog.csdnimg.cn/img_convert/1ad4d7b5478c6f2ec8f9002d3c1186ca.png#align=left&display=inline&height=314&margin=[objectObject]&name=image.png&originHeight=628&originWidth=1549&size=65445&status=done&style=none&width=774.5)
### 2. 修改kubeadm初始化文件
```html
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.3.2.5
  bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  name: sh-master-01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
  certSANs:
  - sh-master-01
  - sh-master-02
  - sh-master-03
  - sh-master.k8s.io
  - localhost
  - 127.0.0.1
  - 10.3.2.5
  - 10.3.2.13
  - 10.3.2.16
  - 10.3.2.12
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.3.2.12:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.5
networking:
  dnsDomain: xx.daemon
  serviceSubnet: 172.254.0.0/16
  podSubnet: 172.3.0.0/16
scheduler: {}
```
修改的地方在下图中做了标识
![image.png](https://img-blog.csdnimg.cn/img_convert/6704923aff5211e9f6abbaef7dcfb405.png#align=left&display=inline&height=401&margin=[objectObject]&name=image.png&originHeight=801&originWidth=1461&size=79487&status=done&style=none&width=730.5)
### 3. kubeadm master-01节点初始化（屏蔽kube-proxy）。
```html
kubeadm init --skip-phases=addon/kube-proxy --config=config.yaml
```
安装成功截图就忽略了，后写的笔记没有保存截图。成功的日志中包含 
```
mkdir -p $HOME/.kube
mkdir -p $HOME/.kube  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

按照输出sh-master-02 ，sh-master-03节点加入集群
将sh-master-01 /etc/kubernetes/pki目录下ca.* sa.* front-proxy-ca.* etcd/ca* 打包分发到sh-master-02,sh-master-03 /etc/kubernetes/pki目录下 
kubeadm join 10.3.2.12:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:eb0fe00b59fa27f82c62c91def14ba294f838cd0731c91d0d9c619fe781286b6     --control-plane
然后同sh-master-01一样执行一遍下面的命令：
mkdir -p $HOME/.kube
sudo \cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


# 3. helm 安装 部署cilium 与hubble(默认helm3了)
## 1. 下载helm并安装helm
注： 由于网络原因。下载helm安装包下载不动经常，直接github下载到本地了
![image.png](https://img-blog.csdnimg.cn/img_convert/efff9f32fe36c47fb54983fdb90e143f.png#align=left&display=inline&height=453&margin=[objectObject]&name=image.png&originHeight=906&originWidth=1504&size=124812&status=done&style=none&width=752)
```
tar zxvf helm-v3.5.3-linux-amd64.tar.gz 
cp helm /usr/bin/

```
## 2 . helm 安装cilium hubble
早先版本 cilium 与hubble是分开的现在貌似都集成了一波流走一遍：
```
helm install cilium cilium/cilium --version 1.9.5
--namespace kube-system
--set nodeinit.enabled=true
--set externalIPs.enabled=true
--set nodePort.enabled=true
--set hostPort.enabled=true
--set pullPolicy=IfNotPresent
--set config.ipam=cluster-pool
--set hubble.enabled=true
--set hubble.listenAddress=":4244"
--set hubble.relay.enabled=true
--set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"
--set prometheus.enabled=true
--set peratorPrometheus.enabled=true
--set hubble.ui.enabled=true
--set kubeProxyReplacement=strict
--set k8sServiceHost=10.3.2.12
--set k8sServicePort=6443
```
部署成功就是这样的
![image.png](https://img-blog.csdnimg.cn/img_convert/50bc9d6e6dbf7d1216ce4788fb073a4f.png#align=left&display=inline&height=35&margin=[objectObject]&name=image.png&originHeight=70&originWidth=1379&size=10597&status=done&style=none&width=689.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/8ccf1c18b6d56b745c022252fbd2485f.png#align=left&display=inline&height=262&margin=[objectObject]&name=image.png&originHeight=523&originWidth=1139&size=69949&status=done&style=none&width=569.5)
嗯 木有kube-proxy的（截图是work加点加入后的故node-init  cilium pod都有6个）
# 4. work节点部署
sh-work-01 sh-work-02 sh-work-03节点加入集群
```
kubeadm join 10.3.2.12:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:eb0fe00b59fa27f82c62c91def14ba294f838cd0731c91d0d9c619fe781286b6
```




# 5. master节点验证
随便一台master节点 。默认master-01节点
![image.png](https://img-blog.csdnimg.cn/img_convert/8536796553c4fe440b4f7d601fc1b267.png#align=left&display=inline&height=77&margin=[objectObject]&name=image.png&originHeight=153&originWidth=1389&size=21726&status=done&style=none&width=694.5)
容易出错 的地方

1. 关于slb绑定。绑定一台server然后kubeadm init是容易出差的 slb 端口与主机端口一样。自己连自己是不可以的....不明觉厉。试了好几次。最后绑定三个都先启动了haproxy。
1. cilium依赖于BPF要先确认下系统是否挂载了BPF文件系统（我的是检查了默认启用了）
```
[root@sh-master-01 manifests]# mount |grep bpf
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```


3.关于kubernetes的配置Cgroup设置与containerd一直都用了system,记得检查
```
KUBELET_EXTRA_ARGS= --cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=/run/containerd/containerd.sock

```


4. 在 kube-controller-manager 中使能 PodCIDR

      在 controller-manager.config 中添加`--allocate-node-cidrs=true`
# 6. 其他
## 1. 验证下hubble  hubble ui 
![image.png](https://img-blog.csdnimg.cn/img_convert/cac7069e89112339cda22cdb8f861eba.png#align=left&display=inline&height=54&margin=[objectObject]&name=image.png&originHeight=108&originWidth=838&size=16649&status=done&style=none&width=419)
```
 kubectl edit svc hubble-ui -n kube-system
```
修改为NodePort 先测试一下。后面会用traefik代理![image.png](https://img-blog.csdnimg.cn/img_convert/a503827bd298020697e8201a417d5637.png#align=left&display=inline&height=331&margin=[objectObject]&name=image.png&originHeight=662&originWidth=1303&size=54156&status=done&style=none&width=651.5)
work  or master节点随便一个公网IP+nodeport访问
![image.png](https://img-blog.csdnimg.cn/img_convert/07f177e213fac20911f311a0258f7d66.png#align=left&display=inline&height=454&margin=[objectObject]&name=image.png&originHeight=907&originWidth=1881&size=149033&status=done&style=none&width=940.5)
## 2 .将ETCDCTL工具部署在容器外
很多时候要用etcdctl还要进入容器 比较麻烦，把etcdctl工具直接提取到master01节点docker有copy的命令 containerd不会玩了 直接github仓库下载etcdctl
![image.png](https://img-blog.csdnimg.cn/img_convert/49661639fe9f0f913151bafa10e9bf0f.png#align=left&display=inline&height=391&margin=[objectObject]&name=image.png&originHeight=782&originWidth=1525&size=106981&status=done&style=none&width=762.5)
```
tar zxvf etcd-v3.4.15-linux-amd64.tar.gz
 cd etcd-v3.4.15-linux-amd64/
cp etcdctl /usr/local/bin/etcdctl

cat >/etc/profile.d/etcd.sh<<'EOF'
ETCD_CERET_DIR=/etc/kubernetes/pki/etcd/
ETCD_CA_FILE=ca.crt
ETCD_KEY_FILE=healthcheck-client.key
ETCD_CERT_FILE=healthcheck-client.crt
ETCD_EP=https://10.3.2.5:2379,https://10.3.2.13:2379,https://10.3.2.16:2379

alias etcd_v3="ETCDCTL_API=3 \
    etcdctl   \
   --cert ${ETCD_CERET_DIR}/${ETCD_CERT_FILE} \
   --key ${ETCD_CERET_DIR}/${ETCD_KEY_FILE} \
   --cacert ${ETCD_CERET_DIR}/${ETCD_CA_FILE} \
   --endpoints $ETCD_EP"
EOF
source  /etc/profile.d/etcd.sh
```
验证etcd
etcd_v3 endpoint status --write-out=table
![image.png](https://img-blog.csdnimg.cn/img_convert/c0ca804eb97d20151f29db4f78e025ae.png#align=left&display=inline&height=198&margin=[objectObject]&name=image.png&originHeight=396&originWidth=1308&size=54568&status=done&style=none&width=654)
# 总结
综合以上。基本环境算是安装完了，由于文章是后写的，可能有些地方没有写清楚，想起来了再补呢




