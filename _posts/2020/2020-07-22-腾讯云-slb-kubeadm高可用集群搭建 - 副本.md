---
layout: "post"
title: "2020-07-22-腾讯云-slb-kubeadm高可用集群搭建"
date: "2020-07-22 16:00:00"
category: "kubernetes"
tags:  "kubernetes1.18.6 kuberadm 高可用 ha"
author: duiniwukenaihe
---
* content
{:toc}

 

集群配置：
centos7.7 64位

|  ip              |      主机名        | 
|  :----:         |     :----:        |   
|10.0.4.20        |      vip |  
|10.0.4.27        |  sh-master-01     |
|10.0.4.46        |  sh-master-02     |  
|10.0.4.47        |  sh-master-02     |  
|10.0.4.14        |  sh-node-01       |  
|10.0.4.2         |  sh-node-02       |  
|10.0.4.6         |  sh-node-03       |
|10.0.4.4         |  sh-node-04       |
|10.0.4.13        |  sh-node-05       |


## 背景

 1. 线上稳定跑着1.16.8版本kubeadm高可用ha kubernets集群。三台master节点，配置为4核心8G，slb+haproxy 代理6443实现高可用。work节点为5台8核心16g,主要跑了60多个应用300个左右容器。
 2. 集群采用了slb代理work节点80 443等端口然后用traefik对外暴露应用。日志采集使用了elastic on kubernetes（eck）收集集群日志保留7天内应用日志。另外还搭建了springboot对外收集前端post埋点日志，入kafka。logstash消费入elasticsearch。kibana展示数据。报警监控系统使用了promethus-oprator，报警alartmanager,企业微信报警。granfna展示。持久化存储开始搭建了rook-ceph1.1集群， 但是在版本升级还有节点异常时出现了各种问题，最终放弃。包括eck等应用都使用了local-storage方式存储，elasticsearch的备份使用了腾讯云对象存储服务cos，定制了elasticsearch镜像添加了相关组件。
 3. 项目的更新发布使用了jenkins，集成kubernets。线上环境已经正常运行近一年时间。
 4. 想体验下新版本，然后又动手搭建了一套1.18.6测试环境。中间犯了好多错误，比如iptables没有关闭，也更加深入了解了下负载均衡slb代理本地端口的过程。
 5. 大致过程与1.16差不多，自己写下日志记录一遍。然后今年想深入集成一下腾讯云的cbs.不要问我为什么不用腾讯云的tke.首先每个slb到现在应该还是只可以挂载一个ssl证书的，业务比较少，我不想管理多个负载均衡，然后负载均衡的策略也比较坑，偶尔tke集群slb测还经常更新。还有上传文件大小限制这样的策略。使用traefik的tcp代理很方便解决这些问题。而且我还是比较喜欢原生不喜欢定制。

6. 关于环境的初始化和安装可以看下张馆长的文档，真心不错 https://zhangguanzhang.github.io/2019/11/24/kubeadm-base-use/。


### 一 .首先的还是环境初始化，master work节点全部执行

#### 1. 默认主机名已经与集群配置中对应，hostnamectl  set-hostname设置对应主机名（10.0.4.20为slb负载均衡ip）
#### 2. 升级linux内核
 ```bash
centos7默认内核为3.10版本，一般是建议把内核更新一下。
##导入key
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
##添加elrepo源
rpm -ivh https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
##查看可更新kernel版本
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
##关于kernel的版本 ml（mainline，主线最新版）  lt（长期支持版本）可参照https://www.cnblogs.com/clsn/p/10925653.html。
## 安装长期支持版本
yum --enablerepo=elrepo-kernel -y install kernel-lt
## 查看grub2启动选择项
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
##修改grub2.conf使内核生效
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
##验证内核
uname -a 
##删除旧内核
package-cleanup --oldkernels
 ```
 ![kernel1](/assets/images/2020/07/kubernetes1.18.6/kernel1.png)
 ![kernel2](/assets/images/2020/07/kubernetes1.18.6/kernel2.png)
 ![kernel3](/assets/images/2020/07/kubernetes1.18.6/kernel3.png)
 ![kerne41](/assets/images/2020/07/kubernetes1.18.6/kernel4.png)
#### 3. 关闭swap交换分区
 ```bash
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
 ```
#### 4. 关闭selinux
 ```bash
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config 
 ```
#### 5.  调整文件打开数等配置
 ```bash
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
 ```
#### 6. 开启ip转发优化 网桥等配置
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
 注意：由于kube-proxy使用ipvs的话为了防止timeout需要设置下tcp参数
  ```bash
cat <<EOF >> /etc/sysctl.d/k8s.conf
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
EOF
sysctl --system
 ```
#### 7. 加载ipvs
 ```bash

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
启动该模块管理服务
systemctl daemon-reload
systemctl enable --now systemd-modules-load.service
lsmod | grep ip_v
```
![ipvs](/assets/images/2020/07/kubernetes1.18.6/ipvs.png)
#### 8. journal 日志相关避免日志重复搜集，浪费系统资源。修改systemctl启动的最小文件打开数量,关闭ssh反向dns解析.设置清理日志熟虑，最大20m(可根据个人需求设置)
 ```bash
sed -ri 's/^\$ModLoad imjournal/#&/' /etc/rsyslog.conf
sed -ri 's/^\$IMJournalStateFile/#&/' /etc/rsyslog.conf

sed -ri 's/^#(DefaultLimitCORE)=/\1=100000/' /etc/systemd/system.conf
sed -ri 's/^#(DefaultLimitNOFILE)=/\1=100000/' /etc/systemd/system.conf

sed -ri 's/^#(UseDNS )yes/\1no/' /etc/ssh/sshd_config
journalctl --vacuum-size=20M
 ```
#### 9. 配置yum源
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
#### 10. 安装基本服务
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
},
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
添加个日志最多值，否则有的苦了，入坑体验过了。docker要不要开机启动呢？我后面安装rook ceph 开机重新启动了老有错误，因为没有将节点设置为cordon，但是也懒了， 我就没有设置为开机启动。故开机启动后在启动docker了
 ```
#### 11. 安装kubernetes
 ```bash
 #查看yum源中可支持版本
 yum list --showduplicates kubeadm --disableexcludes=kubernetes 

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
##可指定自己要安装的版本
#yum install -y kubelet-1.18.6 kubeadmt-1.18.6  kubectlt-1.18.6  --disableexcludes=kubernetes
systemctl enable kubelet
  ```

### 二 . master节点操作
 ```bash
注：slb内网传统型负载均衡使用了。尝试了两种方式：
 1. slb+haproxy slb 绑定三台master6443代理后端haproxy 8443端口。（kubeadm-config.yaml配置文件中controlPlaneEndpoint: "10.0.4.20:6443"）。
2. keepalived +haproxy 配置slb地址（三台server设置不同权重，kubeadm-config.yaml配置文件中controlPlaneEndpoint: "10.0.4.20:8443"）。
个人来说强迫症 就喜欢6443所以就用了第一种。
 ```
#### 1. master节点安装haproxy（sh-master01 sh-master-02 sh-master03）
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
    default_backend kubernetes
#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend kubernetes           #后端服务器，也就是说访问192.168.255.140:8443会将请求转发到后端的三台，这样就实现了负载均衡
    balance roundrobin               
    server master1  10.0.4.27:6443 check maxconn 2000
    server master2  10.0.4.46:6443 check maxconn 2000
    server master3  10.0.4.47:6443 check maxconn 2000
EOF
 systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy

腾讯云slb负载均衡最终还是用了传统型，监听器tcp 6443代理后端三台haproxy 8443端口
 ```
  ![slb](/assets/images/2020/07/kubernetes1.18.6/slb.png)
  ![slb1](/assets/images/2020/07/kubernetes1.18.6/slb1.png)
#### 2. kuberadm master安装
 ```bash
master1节点
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  serviceSubnet: "172.251.0.0/16"                         #设置svc网段
  podSubnet: "172.252.0.0/16"                             #设置Pod网段
  dnsDomain: "layabox.sh"
kubernetesVersion: "v1.18.6"                            #设置安装版本
controlPlaneEndpoint: "10.0.4.20:6443"             #设置相关API VIP地址
dns: 
  type: CoreDNS
apiServer:
  certSANs:
  - sh-master-01
  - sh-master-02
  - sh-master-03
  - sh-master.k8s.io
  - 127.0.0.1
  - 10.0.4.27
  - 10.0.4.46
  - 10.0.4.47
  - 10.0.4.20
  timeoutForControlPlane: 4m0s
certificatesDir: "/etc/kubernetes/pki"
imageRepository: "ccr.ccs.tencentyun.com/k8s_containers"  #国内貌似没有最新的镜像库，自己同步到自己镜像仓库了，开始没有将namespace设置为公开，后期无法设置对外，抱歉。
etcd:
    local:
      dataDir: /var/lib/etcd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs  #使用ipvs方式
EOF

kubeadm init --config kubeadm-config.yaml
mkdir -p $HOME/.kube
sudo \cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

按照输出master02 ，master03节点加入集群
将master01 /etc/kubernetes/pki目录下ca.* sa.* front-proxy-ca.* etcd/ca* 打包分发到master02,master03 /etc/kubernetes/pki目录下 
 kubeadm join 10.0.4.20:6443 --token jiprvz.0rkovt1gx3d658j     --discovery-token-ca-cert-hash sha256:5d631bb4bdce033163037ef21f663c88e058e70c6c362c9c5ccb1a92095     --control-plane --certificate-key 
然后同master01一样执行一遍下面的命令：
mkdir -p $HOME/.kube
sudo \cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

 ```
   ![master1](/assets/images/2020/07/kubernetes1.18.6/master1.png)
注： key都胡乱输入的这里没有用自己的。此时任意一台master执行kubectl get nodes  STATUS一列应该都是NOTReady.
#### 3. 配置flannel插件
 ```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
修改配置文件中Network 为自己设置的子网，我这里是172.252.0.0/16
kubectl apply -f kube-flannel.yml
然后基本发现 master节点都已经redeay
 ```
 ### 二 . work节点j加入集群
 ```bash
kubeadm join 192.168.3.9:6443 --token 3o6dy0.9gbbfuf55xiloe9d --discovery-token-ca-cert-hash sha256:5d631bb4bdce01dcad51163037ef21f663c88e058e70c6c362c9c5ccb1a92095
OK集群算是初始搭建完了，不知道跑一遍咋样，我的是正常跑起来了。
 ```
 > 确认集群节点是否ready。常见问题，集群开启了ipvs，但是我iptables没有关闭，然后节点一直加入不了，看了眼防火墙开着呢没有关闭规则。由于主机都是云主机，就开启了安全组策略，把防火墙都关闭了。如果是其他环境一定记得检查防火墙策略。
 
## 集群搭建成功上下图：
  ![status](/assets/images/2020/07/kubernetes1.18.6/status.png)

   ![status1](/assets/images/2020/07/kubernetes1.18.6/status1.png)


## 后记
#### 1. 如果kubeadm-config.yaml配置文件忘了设置ipvs了开启下ipvs.这里记得在
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
3. work节点最好打上标签，给服务设置亲和性和反亲和性。资源的调度使用值貌似可以设置的？否则后面有的work会出现pods一直创建中，打标签合理规划资源还是很有必要的。