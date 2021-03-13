---
layout: "post"
title: "2020-09-11-hadoop-ha-hbase-hive-spark"
date: "2020-09-11 10:00:00"
category: "hadoop"
tags:  "hadoop hbase hive spark"
author: duiniwukenaihe
---
* content
{:toc}

 

集群配置：
centos8 64位

|  ip                                   |      主机名        |        |   服务| 
|  :-------------------------- :        |     :----:        |         |:----:  |
|10.0.4.20                              |      vip |        |      |master zookeeper|
|10.0.4.27                              |  sh-master-01     |      |master zookeeper|
|10.0.4.46                              |  sh-master-02     |      |master zookeeper|
|10.0.4.47                              |  sh-master-02     |  
|10.0.4.14                              |  sh-work-01       |  
|10.0.4.2                               |  sh-work-02       |  
|10.0.4.6                               |  sh-work-03       |



## 背景
>在其他容器环境中其实已正常运行elastic7.6集群稳定运行，但是想把一些数据进行深度挖掘一下，就想跑一下hadoops生态圈。以上几台机器实际上是自己搭建的kubernets1.19.0的centos8集群，算是一个白板集群， 就只集成了腾讯云的cbs硬盘存储storageclass.鉴于个人能力有限，hadoop生态圈不是太了解，就在云主机实体机直接安装了。鸣谢陌生的山不知名的花：https://mshk.top/2019/03/centos-hadoop-zookeeper-hbase-hive-spark-high-availability/。很多地方借鉴了下。其实hadoop3可以使用腾讯云cos云存储了， 但是自己没有整明白，而且hadoop-cos文档写的比较乱，期待有大佬能有写下借鉴的。

环境相关软件版本号：
hadoop3.3.0+apache-zookeeper-3.6.1+hbase-2.2.4+apache-hive-3.1.2+scala-2.13.3+spark-3.0.1
约定：
1. 系统为腾讯云centos8系统，ssh端口为36000自定义。
2. java环境已经安装java8 jdk1.8.0_251版本。
3. ssh-key默认已经在sh-master-01生成并copy到其他节点，已经建立信任关系。
4. 默认软件安装位置为work目录，目录结构定义如下：
> 1. src 软件安装包下载位置
> 2. app 软件运行目录
> 3. logs 软件运行日志目录
> 4. data 软件数据目录





#### 一.环境准备
关于host:
 ```bash
cat /etc/hosts
10.0.4.27 sh-master-01
10.0.4.46 sh-master-02
10.0.4.47 sh-master-03
10.0.4.14 sh-work-01
10.0.4.2  sh-work-02
10.0.4.6  sh-work-03
注： hostnamectl --static set-hostname 设置主机名，此步骤省略，已在kubernets环境中设置
 ```
关于ssh免密码登陆 
 ```bash
 ssh-keygen
 #一路回车
ssh-copy-id sh-master-01 -p 36000
ssh-copy-id sh-master-02 -p 36000
ssh-copy-id sh-master-03 -p 36000

ssh-copy-id sh-work-01 -p 36000
ssh-copy-id sh-work-02 -p 36000
ssh-copy-id sh-work-03 -p 36000
 ```
测试秘钥是否配置成功
 ```bash
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N hostname; done;
sh-master-01
sh-master-02
sh-master-03
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N hostname; done;
sh-work-01
sh-work-02
sh-work-03
 ```
#### 二. 下载应用程序配置环境变量
创建安装目录
  创建要用到的目录结构，所有的程序都统一在/work/app 目录，所有下载的源码在 /home/work/src目录 ，所有的数据在 /home/work/data 目录，所有的日志在 /home/work/logs 目录。
 ```bash
# 创建 Hadoop3.1.2 和 Zookeeper3.4.13 需要的目录
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/{src,app,logs,data} -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /work/{src,app,logs,data} -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/data/{hadoop,zookeeper} -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /work/logs/{hadoop,zookeeper} -p; done;

## 在 Hadoop3.1.2 的 NameNode 上创建 HA 共享目录
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/data/hadoop/{journalnode,ha-name-dir-shared} -p; done;

# 创建 Hbase1.4.9 需要的目录
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/logs/hbase -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /work/logs/hbase -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/data/hbase -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /work/data/hbase -p; done;
# 创建 Hive2.3.4 需要的目录
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /work/data/hive/{scratchdir,tmpdir} -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /work/data/hive/{scratchdir,tmpdir} -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir  /work/logs/hive -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir  /work/logs/hive -p; done;
# 创建 Spark2.4.0 需要的目录
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N mkdir /home/work/data/spark -p; done;
[root@sh-master-01 ~]# for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N mkdir /home/work/logs/spark -p; done;
 ```
什么诡异工具包
yum install net-tools checkpolicy gcc dkms foomatic openssh-server bash-completion psmisc -y
官网下载就好了，吐个糟 zookeeper其他版本有的会有问题， hbase-2.2.5我安装了也诡异换成2.2.4可以了
[root@sh-master-01 _src]# cd /work/src/
[root@sh-master-01 src]# ll
total 1714920
-rw-r--r-- 1 root root 278813748 Sep 11 17:57 apache-hive-3.1.2-bin.tar.gz
-rw-r--r-- 1 root root  12436328 Sep 11 17:57 apache-zookeeper-3.6.1-bin.tar.gz
-rw-r--r-- 1 root root 500749234 Sep 11 17:57 hadoop-3.3.0.tar.gz
-rw-r--r-- 1 root root 223600848 Sep 11 17:57 hbase-2.2.4-bin.tar.gz
-rw-r--r-- 1 root root 220221311 Sep 11 17:57 hbase-2.2.5-bin.tar.gz
-rw-r--r-- 1 root root 271726801 Sep 11 17:57 hbase-2.3.1-bin.tar.gz
-rw-r--r-- 1 root root   1007505 Sep 11 17:57 mysql-connector-java-5.1.47-bin.jar
-rw-r--r-- 1 root root   1007502 Sep 11 17:57 mysql-connector-java-5.1.47.jar
-rw-r--r-- 1 root root  22414008 Sep 11 17:57 scala-2.13.3.tgz
-rw-r--r-- 1 root root 224062525 Sep 11 17:57 spark-3.0.1-bin-hadoop3.2.tgz








for N in $(seq 2 3); do scp -P 36000 -r /work/app/zookeeper sh-master-0$N:/work/app/; done;

for N in $(seq 2 3); do scp -P 36000 -r /work/app/hadoop sh-master-0$N:/work/app/; done;
for N in $(seq 1 3); do scp -P 36000 -r /work/app/hadoop sh-work-0$N:/work/app/; done;

for N in $(seq 1 3); do ssh -p 36000 sh-master-0$N hdfs --daemon start journalnode;jps; done;
for N in $(seq 1 3); do ssh -p 36000 sh-work-0$N hdfs --daemon start journalnode;jps; done;

hdfs namenode -format



hdfs zkfc -formatZK
hdfs --daemon start namenode

hdfs namenode -bootstrapStandby
hdfs --daemon start namenode


[root@sh-master-01 hadoop]# hdfs haadmin -getAllServiceState
sh-master-01:8020                                  standby   
sh-master-02:8020                                  standby   
sh-master-03:8020                                  standby

start-dfs.sh


 1840  2020-09-12 10:06:09 for N in $(seq 1 3); do scp -P 36000 -r /work/app/hadoop/etc sh-work-0$N:/work/app/hadoop; done;
 1841  2020-09-12 10:06:49 for N in $(seq 2 3); do scp -P 36000 -r /work/app/hadoop/etc sh-master-0$N:/work/app/hadoop; done;



hdfs haadmin -getAllServiceState

start-yarn.sh 
yarn jar /work/app/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.0.jar wordcount /mshk.top/test.mshk.top.txt /output

hadoop fs -cat /output/part-*

 mapred --daemon start historyserver


for N in $(seq 2 3); do scp -P 36000 -r /work/app/hbase sh-master-0$N:/work/app/; done;
for N in $(seq 1 3); do scp -P 36000 -r /work/app/hbase sh-work-0$N:/work/app/; done;
start-hbase.sh 

master02 master-03
hbase-daemon.sh start master

在讲述**前置要求**之前，对于各组件设置参数启动项有些要注意的地方：
- 有些feature gates在GA以后的版本不能再被显式设置，否则可能导致报错。实际上这些feature gates在beta版本开始则无需添加。下表整理了涉及到feature gates的beta版本的表格，在给kubelet、master/controllermanager、scheduler设置启动参数时，可以基于此来做取舍.（举例：KubeletPluginsWatcher在1.12及以上版本则无须添加）。


| 特性                         | 默认值    | 阶段   | 起始   | 直到   |
| -------------------------- | ------ | ---- | ---- | ---- |
| `VolumeSnapshotDataSource` | `true` | Beta | 1.17 | -    |
| `CSINodeInfo`              | `true` | Beta | 1.14 | 1.16 |
| `CSIDriverRegistry`        | `true` | Beta | 1.14 | 1.17 |
| `KubeletPluginsWatcher`    | `true` | Beta | 1.12 | 1.12 |
| `VolumeScheduling`         | `true` | Beta | 1.10 | 1.12 |




#### 三. 配置
**注意**: https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_COSFS.md 文档比较坑，首先 rbac都没有搞，照着下去能成功吗？反正我搞东西 先rbac授权。
##### 1. 配置rbac

 ```
#参考deploy/cosfs/kubernetes/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-cos-tencentcloud
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-cos-tencentcloud
rules:
  - apiGroups: [""]
    resources: ["events", "persistentvolumes"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-cos-tencentcloud
subjects:
  - kind: ServiceAccount
    name: csi-cos-tencentcloud
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: csi-cos-tencentcloud
  apiGroup: rbac.authorization.k8s.io 
 ```


> kubectl apply -f rbac.yaml

##### 2. Deploy CSI external components 翻译过来是叫部署容器存储接口（csi） 外部组件？
 ```
#参考deploy/cosfs/kubernetes/cosattacher.yaml
kind: Service
apiVersion: v1
metadata:
  name: csi-cosplugin-external-runner
  namespace: kube-system
  labels:
    app: csi-cosplugin-external-runner
spec:
  selector:
    app: csi-cosplugin-external-runner
  ports:
    - name: dummy
      port: 12345

---
kind: StatefulSet
apiVersion: apps/v1
metadata:
  name: csi-cosplugin-external-runner
  namespace: kube-system
spec:
  serviceName: "csi-cosplugin-external-runner"
  replicas: 1
  selector:
    matchLabels:
      app: csi-cosplugin-external-runner
  template:
    metadata:
      labels:
        app: csi-cosplugin-external-runner
    spec:
      serviceAccount: csi-cos-tencentcloud
      containers:
        - name: csi-attacher
          image: ccr.ccs.tencentyun.com/ccs-dev/csi-attacher:1.0.1
          args:
            - "--v=3"
            - "--csi-address=$(ADDRESS)"
          env:
            - name: ADDRESS
              value: /var/lib/csi/sockets/pluginproxy/csi.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /var/lib/csi/sockets/pluginproxy/
        - name: cosfs
          image: ccr.ccs.tencentyun.com/ccs-dev/csi-tencentcloud-cos:1.0.0
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CSI_ENDPOINT
              value: unix://var/lib/csi/sockets/pluginproxy/csi.sock
          imagePullPolicy: "Always"
          volumeMounts:
            - name: socket-dir
              mountPath: /var/lib/csi/sockets/pluginproxy
      volumes:
        - name: socket-dir
          emptyDir: {}
``` 
```           
原文档 apps/v1beta1 修改为apps/v1 增加
  selector:
    matchLabels:
      app: csi-cosplugin-external-runner
``` 

> kubectl apply -f cosattacher.yaml

![cosattacher](/assets/images/2020/07/cosfs/cosattacher.png) 
![cosattacher-pods](/assets/images/2020/07/cosfs/cosattacher-pods.png) 
 
##### 3.Deploy COS CSI launcher components:部署容器存储接口（csi） 启动组件？ launcher=启动器？
 ``` 
 #参考deploy/cosfs/kubernetes/coslauncher.yaml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: csi-coslauncher
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-coslauncher
  template:
    metadata:
      labels:
        app: csi-coslauncher
    spec:
      hostNetwork: true
      containers:
        - name: cos-launcher
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: ccr.ccs.tencentyun.com/ccs-dev/csi-tencentcloud-cos-launcher:1.0.0
          imagePullPolicy: "Always"
          volumeMounts:
          - name: launcher-socket-dir
            mountPath: /tmp
          - name: pods-mount-dir
            mountPath: /var/lib/kubelet/pods
            mountPropagation: "Bidirectional"
          - mountPath: /dev
            name: host-dev
          - mountPath: /sys
            name: host-sys
          - mountPath: /lib/modules
            name: lib-modules
            readOnly: true
      volumes:
      - name: launcher-socket-dir
        hostPath:
          path: /etc/csi-cos
          type: DirectoryOrCreate
      - name: pods-mount-dir
        hostPath:
          path: /var/lib/kubelet/pods
          type: Directory
      - name: host-dev
        hostPath:
          path: /dev
      - name: host-sys
        hostPath:
          path: /sys
      - name: lib-modules
        hostPath:
          path: /lib/modules
 ```     
  ```       
原文档 apps/v1beta1 修改为apps/v1
 ```  
> kubectl apply -f  coslauncher.yaml
 ![coslauncher](/assets/images/2020/07/cosfs/coslauncher.png) 
![coslauncher-pods](/assets/images/2020/07/cosfs/coslauncher-pods.png) 

##### 4.Deploy COS CSI driver 部署cos  csi驱动
 ```   
#参考deploy/cosfs/kubernetes/cosplugin.yaml
# This YAML file contains driver-registrar & csi driver nodeplugin API objects,
# which are necessary to run csi nodeplugin for oss.

kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: csi-cosplugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-cosplugin
  template:
    metadata:
      labels:
        app: csi-cosplugin
    spec:
      serviceAccount: csi-cos-tencentcloud
      hostNetwork: true
      containers:
        - name: driver-registrar
          image: ccr.ccs.tencentyun.com/ccs-dev/csi-node-driver-registrar:1.0.2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "rm -rf /registration/com.tencent.cloud.csi.cosfs /registration/com.tencent.cloud.csi.cosfs-reg.sock"]
          args:
            - "--v=5"
            - "--csi-address=$(ADDRESS)"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/com.tencent.cloud.csi.cosfs/csi.sock"
          env:
            - name: ADDRESS
              value: /plugin/csi.sock
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: plugin-dir
              mountPath: /plugin
            - name: registration-dir
              mountPath: /registration
        - name: cosfs
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: ccr.ccs.tencentyun.com/ccs-dev/csi-tencentcloud-cos:1.0.0
          args:
            - "--nodeID=$(NODE_ID)"
            - "--endpoint=$(CSI_ENDPOINT)"
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CSI_ENDPOINT
              value: unix://plugin/csi.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: plugin-dir
              mountPath: /plugin
            - name: launcher-socket-dir
              mountPath: /tmp
      volumes:
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins/com.tencent.cloud.csi.cosfs
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: Directory
        - name: launcher-socket-dir
          hostPath:
            path: /etc/csi-cos
            type: DirectoryOrCreate
 ```   
  ```   
 原文档 apps/v1beta2 修改为apps/v1
  ```  
> kubectl apply -f  cosplugin.yaml  


![cosplugin-pods](/assets/images/2020/07/cosfs/cosplugin-pods.png) 


#### 四. DEMO


##### 1. create a secret 创建secret秘钥

  ``` 
#参考deploy/cosfs/examples/secret.yaml  
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  # Replaced by your secret name.
  name: cos-secret
  # Replaced by your secret namespace.
  namespace: kube-system
data:
  # Replaced by your temporary secret file content. You can generate a temporary secret key with these docs:
  # Note: The value must be encoded by base64.
  SecretId: VWVEJxRk5Fb0JGbDA4M...(base64 encode)
  SecretKey: Qa3p4ZTVCMFlQek...(base64 encode)
```  
```  
kubectl apply -f secret 
or 
创建secret的另外一种方式：kubectl create secret generic cos-secret -n kube-system  --from-literal=SecretId=AKIDjustfortest --from-literal=SecretKey=justfortest
```  
![cos-secret](/assets/images/2020/07/cosfs/cos-secret.png) 
##### 2. create  pv and pvc 创建pv pvc 

```  
###参考deploy/cosfs/examples/pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pv-cos"
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  csi:
    driver: com.tencent.cloud.csi.cosfs
    # Specify a unique volumeHandle like pv name or bucket name
    volumeHandle: pv-cos
    volumeAttributes:
      # Replaced by the url of your region.
      url: "http://cos.ap-shanghai.myqcloud.com"
      # cosfs log level, will use node syslog, support [dbg|info|warn|err|crit]
      dbglevel: "err"
      # Replaced by the bucket name you want to use.
      bucket: "kubernetes-xxxxxx"
      # You can specify any other options used by the cosfs command in here.
      additional_args: "-oensure_diskfree=20480"
    nodePublishSecretRef:
      # Replaced by the name and namespace of your secret.
      name: cos-secret
      namespace: kube-system
###参考deploy/cosfs/examples/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: pvc-cos
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  # You can specify the pv name manually or just let kubernetes to bind the pv and pvc.
  # volumeName: pv-cos
  # Currently cos only supports static provisioning, the StorageClass name should be empty.
  storageClassName: ""  
```   
```  
kubectl apply -f pv.yaml

kubectl apply -f pvc.yaml

 ```       

 
![pv-pvc1](/assets/images/2020/07/cosfs/pv-pvc1.png) 
![pv-pvc](/assets/images/2020/07/cosfs/pv-pvc.png)
##### 4. Create a Pod to use the PVC
```  
#参考deploy/cosfs/examples/podyaml
apiVersion: v1
kind: Pod
metadata:
    name: pod-cos
spec:
  containers:
  - name: pod-cos
    command: ["tail", "-f", "/etc/hosts"]
    image: "centos:latest"
    volumeMounts:
    - mountPath: /data
      name: cos
    resources:
      requests:
        memory: "128Mi"
        cpu: "0.1"
  volumes:
  - name: cos
    persistentVolumeClaim:
      # Replaced by your pvc name.
      claimName: pvc-cos
``` 

> kubectl apply -f pod.yaml

![cos-pods](/assets/images/2020/07/cosfs/cos-pods.png)
![pod-touch](/assets/images/2020/07/cosfs/pod-touch.png)
![cos](/assets/images/2020/07/cosfs/cos.png)