---
layout: "post"
title: "2019-11-26-k8s-install-now"
date: "2019-11-26 10:00:00"
category: "kubernetes"
tags:  "kubernetes1.16.2 kuberadm"
author: duiniwukenaihe
---
* content
{:toc}

 

集群配置：
初始集群环境kubeadm 1.16.2

|  ip           | 自定义域名         |    主机名 |
|  :----:       |     :----:        |   :----:  |
|192.168.3.9      |  master.k8s.io    |  k8s-vip  |
|192.168.3.10    |  master01.k8s.io  |  k8s-master-01|
|192.168.3.5   |  master02.k8s.io  |  k8s-master-02| 
|192.168.3.12   |  master03.k8s.io  |  k8s-master-03|
|192.168.3.6    |  node01.k8s.io    |  k8s-node-01|
|192.168.3.2    |  node02.k8s.io    |  k8s-node-02|
|192.168.3.4    |  node03.k8s.io    |  k8s-node-03|

## 背景

>前面192.168.20.13这几台机器是用kubeadm1.15 搭建过 kubernetes的，后续出现了很多问题。开始的规划很不完善，后面就重新搭建了记录下：首先说下原来的不满意的地方：
>
1. etcd自建外部挂载，个人对etcd不是很懂，版本升级兼容问题各种解决毕竟费劲，更主要的是都上容器了，我为什么不把etcd教给容器呢？当然了存储还是挂载master主机目录的。
2. 腾讯云的slb了 还使用了haproxy，开始使用应用型负载均衡代理，后而且后面出现了各种诡异的问题，比如证书之类的。个人觉得问题应该简单化。

## 开始

### 首先的还是环境初始化，master work节点全部执行
> 默认主机名已经与集群配置中对应，hostnamectl  set-hostname设置过主机名
#### 1. 关闭swap
 ```bash
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
 ```
#### 2. 关闭selinux
 ```bash
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config 
 ```bash
#### 3. 调整文件打开数等配置
 ```bash
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
 ```
#### 4. 开启ip转发
 ```bash
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
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
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
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
 ```

#### 5. 加载ipvs
 ```bash
vim /etc/sysconfig/modules/ipvs.modules
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
yum install ipset
 ```
#### 6. journal 日志相关这里因为后面吃亏了 日志没有做切割保存，查看问题太麻烦了
 ```bash
sed -ri 's/^\$ModLoad imjournal/#&/' /etc/rsyslog.conf
sed -ri 's/^\$IMJournalStateFile/#&/' /etc/rsyslog.conf

sed -ri 's/^#(DefaultLimitCORE)=/\1=100000/' /etc/systemd/system.conf
sed -ri 's/^#(DefaultLimitNOFILE)=/\1=100000/' /etc/systemd/system.conf

sed -ri 's/^#(UseDNS )yes/\1no/' /etc/ssh/sshd_config
journalctl --vacuum-size=20M
 ```
#### 7. 配置yum源
 ```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg 
EOF
 ```
#### 8. 安装基本服务
 ```bash
安装依赖包
yum install -y epel-release
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl
安装bash命令提示
yum install -y bash-argsparse bash-completion bash-#completion-extras
安装docker kubeadm:
yum install docker-ce -y
#配置镜像加速器 
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://lrpol8ec.mirror.aliyuncs.com"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
}
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
添加个日志最多值，否则有的苦了，入坑体验过了。docker要不要开机启动呢？我后面安装rook ceph 开机重新启动了老有错误，因为没有将节点设置为cordon，但是也懒了， 我就没有设置为开机启动。故开机启动后在启动docker了
 ```
#### 9. 安装kubernetes
 ```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet
  ```

###  master节点操作
#### 1. master节点安装haproxy
 ```bash
yum install -y haproxy
cat <<EOF > /etc/haproxy/haproxy.cfg

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
    option                  httplog
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
    default_backend kubernetes-master
#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend kubernetes-master           #后端服务器，也就是说访问192.168.255.140:8443会将请求转发到后端的三台，这样就实现了负载均衡
    balance roundrobin               
    server master1  192.168.3.10:6443 check maxconn 2000
    server master2  192.168.3.5:6443 check maxconn 2000
    server master3  192.168.3.12:6443 check maxconn 2000
EOF
 systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy

腾讯云lsb负载均衡最终还是用了传统型，监听器tcp 6443代理后端三台haproxy 8443端口
 ```
#### 2. kuberadm master安装
 ```bash
master1节点
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.2
apiServer:
  certSANs:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - k8s-master-04
    - master.k8s.io
    - 192.168.3.10
    - 192.168.3.5
    - 192.168.3.12
    - 192.168.3.9
    - 192.168.3.3
    - 127.0.0.1
controlPlaneEndpoint: "192.168.3.9:6443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
    local:
      dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
networking:
  podSubnet: 10.30.0.0/16
  serviceSubnet: 10.31.0.0/16
EOF
kubectl apply -f kubeadm-config.yaml
现在最新的是1.16.3，安装的时候是1.16.2就用了默认配置文件了。网络规划不会弄，这样貌似有点很差劲，因为以后想弄联邦集群，后面再想解决方法吧。另外腾讯云曾经开源过一个tencentcloud-cloud-controller-manager，其实很多可以打通的，但是试用了下 坑多的样子没有跑通，放弃了
kubeadm init --config initconfig.yaml
mkdir -p $HOME/.kube
sudo \cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

按照输出master02 ，master03节点加入集群
 kubeadm join 192.168.3.9:6443 --token jiprvz.0rkovt1gx3d658j     --discovery-token-ca-cert-hash sha256:5d631bb4bdce033163037ef21f663c88e058e70c6c362c9c5ccb1a92095     --control-plane --certificate-key 0eaa7e5f8efbdc8d381fb329c28c49f87af284fecc0c9443501e81f3cdc4
将master01 /etc/kubernetes/pki目录下ca* sa* fr* etcd 打包分发到master02,master03 /etc/kubernetes/pki目录下 
注： key都胡乱输入的这里没有用自己的，复制pki这部忘了 老的版本都复制来，记得这个版本我没有复制key的？可以安装流程自己看看
 ```
#### 3. 配置flannel插件
 ```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
修改配置文件中Network 为自己设置的子网，我这里是10.30.0.0/16
kubectl apply -f kube-flannel.yml
然后基本发现 master节点都已经redeay
 ```
#### 4. work节点加入master
 ```bash
kubeadm join 192.168.3.9:6443 --token 3o6dy0.9gbbfuf55xiloe9d --discovery-token-ca-cert-hash sha256:5d631bb4bdce01dcad51163037ef21f663c88e058e70c6c362c9c5ccb1a92095
OK集群算是初始搭建完了，不知道跑一遍咋样，我的是正常跑起来了。
 ```
#### 5. 配置文件忘了设置ipvs了开启下ipvs.这里记得在
 ```bash
kubectl edit cm kube-proxy -n kube-system
configmap/kube-proxy edited

#修改如下
kind: MasterConfiguration
apiVersion: kubeadm.k8s.io/v1alpha1
...
ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      syncPeriod: 30s
    kind: KubeProxyConfiguration
    metricsBindAddress: 127.0.0.1:10249
    mode: "ipvs"                  #修改

kubectl get pod -n kube-system | grep kube-proxy |awk '{system("kubectl delete pod "$1" -n kube-system")}'
 ```
> 貌似应该就跑起来了，然后后面应该还要做的：
1. etcd的备份，虽然有三个master节点 数据无价，还是做下etcd的备份要好。
2. pods 可能都running了 但是最后还是看下日志，肯能有些小的失误，看日志是个好习惯的，老版本糊里糊涂搭建的时候kubernetes插件pod打了一大堆日志 虽然可以使用，但是还是要追求下完美的。由此可见搭建日志采集系统还是很有必要的。
3. work节点最好打上标签，不是服务设置亲和性和反亲和性。资源的调度使用值貌似可以设置的？否则后面有的work会出现pods一直创建中，打标签合理规划资源还是很有必要的。