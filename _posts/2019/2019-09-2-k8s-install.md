---
layout: post
title: "腾讯云高可用k8s环境安装"
date: "2019-09-02 16:00:00"
category: kubernetes
tags: kubernetes 
author: duiniwukenaihe
---
* content
{:toc}

腾讯云高可用 k8s 集群安装文档  
注： 参考https://blog.csdn.net/qq_32641153/article/details/90413640

**集群环境：**


|  ip           | 自定义域名         |    主机名 |
|  :----:       |     :----:        |   :----:  |
|192.168.20.13  |  master.k8s.io    |  k8s-vip  |
|192.168.2.8    |  master01.k8s.io  |  k8s-master-01|
|192.168.2.12   |  master02.k8s.io  |  k8s-master-02| 
|192.168.2.6    |  master03.k8s.io  |  k8s-master-03|
|192.168.2.3    |  node01.k8s.io    |  k8s-node-01|
|192.168.2.9    |  node02.k8s.io    |  k8s-node-02|

host如下图：

![k8s-host.png](/assets/images/k8s/k8s-host.png)


  ```bash 
注： 防火墙 selinux  swap   默认系统镜像已经关闭 最大文件开启个数等已经优化
#swap
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
#selinux
setenforce  0 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/sysconfig/selinux 
sed -i "s/^SELINUX=permissive/SELINUX=disabled/g" /etc/selinux/config 

#调整文件打开数等配置
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
 

```
--------------------- 



**修改主机名：**

如下图所示，以此修改主机名:

![set-hostname.png](/assets/images/k8s/set-hostname.png)

  ```bash 
  ##修改 192.168.2.8 服务器
hostnamectl  set-hostname  k8s-master-01
  ##修改 192.168.2.12 服务器
hostnamectl  set-hostname  k8s-master-02
  ##修改 192.168.2.6 服务器
hostnamectl  set-hostname  k8s-master-03
  ##修改 192.168.2.3 服务器
hostnamectl  set-hostname  k8s-node-01
  ##修改 192.168.2.9 服务器
hostnamectl  set-hostname  k8s-node-02
```

**系统参数调优,开启ip转发刷新加载配置:**
```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
![k8s-host.png](/assets/images/k8s/ipv4.png)
```bash
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
sysctl -p /etc/sysctl.d/k8s.conf
```
![k8s-host.png](/assets/images/k8s/reshen_ipv4.png)

**配置docker kubernetes yum 源：**


```bash  
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```  

```bash 
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
**开始安装基本服务：**
```bash
安装依赖包
yum install -y epel-release
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl
安装bash命令提示
yum install -y bash-argsparse bash-completion bash-completion-extras

```

```bash
开启ipvs
vim /etc/sysconfig/modules/ipvs.modules
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

yum install ipset
```
**安装docker kubeadm:**

```bash
yum install docker-ce -y
#配置镜像加速器 
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://lrpol8ec.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

![k8s-host.png](/assets/images/k8s/docker_boost.png)

**安装kubelet  kubeadm  组件: ** 

#默认最新版本 可选择版本参数
```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet
```

##安装etcd

默认三台master安装etcd集群


使用cfssl 生产自签名证书 

注：master1执行.
```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
cd /root/ssl

cat <<EOF > ca-config.json 
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
cat <<EOF > ca-csr.json 
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

cat <<EOF > server-csr.json 
{
    "CN": "etcd",
    "hosts": [
    "192.168.20.8",
    "192.168.20.12",
    "192.168.20.6"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

ls *pem

```
![k8s-host.png](/assets/images/k8s/cfssl.png)

三台master (etcd节点)

```bash
wget https://github.com/coreos/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz

tar zxvf etcd-v3.3.13-linux-amd64.tar.gz 
cd etcd-v3.3.13-linux-amd64/ 
mv etcd etcdctl /usr/bin/  
mkdir /var/lib/etcd
mkdir /etc/etcd 
mkdir cfg ssl 
vi cfg/etcd 
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.20.8:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.20.8:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.20.8:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.20.8:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.20.8:2380,etcd02=https://192.168.20.12:2380,etcd03=https://192.168.20.6:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
 
master1
cd /root/ssl
cp *.pem /etc/etcd/ssl/

将ssl 证书copy到master2  mater3
cd /etc/systemd/system/
cat etcd.service
[Unit]
Description=etcd server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=/etc/etcd/cfg/etcd
ExecStart=/usr/bin/etcd \
  --name=etcd01 \
  --cert-file=/etc/etcd/ssl/server.pem \
  --key-file=/etc/etcd/ssl/server-key.pem \
  --peer-cert-file=/etc/etcd/ssl/server.pem \
  --peer-key-file=/etc/etcd/ssl/server-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem
  --initial-advertise-peer-urls https://192.168.20.8:2380 \
  --listen-peer-urls https://192.168.20.8:2380 \
  --listen-client-urls https://192.168.20.8:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.20.8:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd01=https://192.168.20.8:2380,etcd02=https://192.168.20.12:2380,etcd03=https://192.168.20.6:2380 \
  --initial-cluster-state new \
  --initial-cluster-token=etcd-cluster \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target

```
![k8s-host.png](/assets/images/k8s/etcd0.png)
![k8s-host.png](/assets/images/k8s/etcd1.png)
![k8s-host.png](/assets/images/k8s/etcd2.png)
![k8s-host.png](/assets/images/k8s/etcd3.png)
![k8s-host.png](/assets/images/k8s/etcd.service.png)
![k8s-host.png](/assets/images/k8s/etcd_start.png)
![k8s-host.png](/assets/images/k8s/etcd-helth.png)

systemctl start etcd 
发现无法启动看日志
annot add dependency job for unit rpcbind.socket, ignoring: Unit not found


果断  yum install rpcbind -y,重启etcd服务

systemctl start etcd

## haproxy 安装:
yum install -y haproxy

```bash 
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
    server master1  192.168.20.8:6443 check maxconn 2000
    server master2  192.168.20.12:6443 check maxconn 2000
    server master3  192.168.20.6:6443 check maxconn 2000
EOF
```

**启动服务**
```bash systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy ```

## kubernetes install
**负载均衡slb配置监听端口：**

注： slb后台配置

![k8s-host.png](/assets/images/k8s/slb1.png)

![k8s-host.png](/assets/images/k8s/slb2.png)

![k8s-host.png](/assets/images/k8s/slb3.png)

**master01 生成 kubeadm-config.yaml：**
```bash 
cat <<EOF > kubeadm-config.yaml
apiServer:
  certSANs:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - master.k8s.io
    - 192.168.2.8
    - 192.168.2.12
    - 192.168.2.6
    - 192.168.2.13
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "master.k8s.io:6443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  external:
    endpoints:
    - https://192.168.20.8:2379
    - https://192.168.20.12:2379
    - https://192.168.20.6:2379
    caFile: /etc/etcd//ssl/ca.pem
    certFile: /etc/etcd/ssl/server.pem
    keyFile: /etc/etcd/ssl/server-key.pem
    local:    
      dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.15.1
networking: 
  dnsDomain: cluster.local  
  podSubnet: 10.20.0.0/16
  serviceSubnet: 10.120.0.0/16
scheduler: {}
EOF
```
![k8s-host.png](/assets/images/k8s/kuberadm-config.png) 

**master01 init安装：**

kubeadm init --config kubeadm-config.yaml 

![k8s-host.png](/assets/images/k8s/kubinit.png)

**配置flannel 网络：** 

wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl  apply -f kube-flannel.yml

![k8s-host.png](/assets/images/k8s/flannel.png)


将master01 /etc/kubernetes/pki目录下ca* sa* fr* 打包分发到master02,master03 /etc/kubernetes/pki目录下 

![k8s-host.png](/assets/images/k8s/copyca.png) 

**master02,master03加入控制节点：**

  kubeadm join master.k8s.io:6443 --token mx1ab0.g5oeh08uvi4dzs26     --discovery-token-ca-cert-hash sha256:2cb8e41a15719139df01d806f7c174289dac14843830f91ebfd7b9307b70acec     --control-plane


**node01,node02节点执行以下命令加入集群：**

kubeadm join master.k8s.io:6443 --token mx1ab0.g5oeh08uvi4dzs26 \
    --discovery-token-ca-cert-hash sha256:2cb8e41a15719139df01d806f7c174289dac14843830f91ebfd7b9307b70acec 


**kubectl get nodes：**

![k8s-host.png](/assets/images/k8s/get-nodes.png) 

注：开始默认安装的是1.15.1 但是get nodes没有截图固显示1.15.3为升级后版本