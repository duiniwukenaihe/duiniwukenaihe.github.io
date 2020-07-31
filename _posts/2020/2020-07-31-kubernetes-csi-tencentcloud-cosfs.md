---
layout: "post"
title: "2020-07-31-kubernetes集群使用腾讯云cos存储"
date: "2020-07-31 10:00:00"
category: "kubernetes"
tags:  "kubernetes1.18.6 kubernetes-csi-tencentcloud cosfs"
author: duiniwukenaihe
---
* content
{:toc}

 

集群配置：
centos7.7 64位

|  ip                                   |      主机名        | 
|  :-------------------------- :        |     :----:        |   
|10.0.4.20                              |      vip |  
|10.0.4.27                              |  sh-master-01     |
|10.0.4.46                              |  sh-master-02     |  
|10.0.4.47                              |  sh-master-02     |  
|10.0.4.14                              |  sh-node-01       |  
|10.0.4.2                               |  sh-node-02       |  
|10.0.4.6                               |  sh-node-03       |
|10.0.4.4                               |  sh-node-04       |
|10.0.4.13                              |  sh-node-05       |


## 背景
>环境为kubernets集群1.18.6，参照 https://duiniwukenaihe.github.io/2020/07/22/tencent-slb-kubeadm-ha/在腾讯云上搭建。参照https://duiniwukenaihe.github.io/2020/07/23/kubernetes-csi-tencentcloud-cbs/完成 cbs云硬盘块存储集成。现在想把cos作为存储试一下。


#### 1.git clone 仓库
 ```bash
https://github.com/TencentCloud/kubernetes-csi-tencentcloud
现在github会非常卡你懂的，最好还是国外服务器下载了。
 ```
#### 2. kubernets集群更改配置

> 参照https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_COSFS.md，我的kubernets集群是1.18.6。按照https://github.com/TencentCloud/kubernetes-csi-tencentcloud/edit/master/docs/README_CBS_zhCN.md的前置特性KubeletPluginsWatcher支持到1.12 我就不去考虑了。如下：

**注意**:

在讲述**前置要求**之前，对于各组件设置参数启动项有些要注意的地方：
- 有些feature gates在GA以后的版本不能再被显式设置，否则可能导致报错。实际上这些feature gates在beta版本开始则无需添加。下表整理了涉及到feature gates的beta版本的表格，在给kubelet、master/controllermanager、scheduler设置启动参数时，可以基于此来做取舍.（举例：KubeletPluginsWatcher在1.12及以上版本则无须添加）。


| 特性                         | 默认值    | 阶段   | 起始   | 直到   |
| -------------------------- | ------ | ---- | ---- | ---- |
| `VolumeSnapshotDataSource` | `true` | Beta | 1.17 | -    |
| `CSINodeInfo`              | `true` | Beta | 1.14 | 1.16 |
| `CSIDriverRegistry`        | `true` | Beta | 1.14 | 1.17 |
| `KubeletPluginsWatcher`    | `true` | Beta | 1.12 | 1.12 |
| `VolumeScheduling`         | `true` | Beta | 1.10 | 1.12 |




#### 2. 配置
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


#### 3. DEMO


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

> kubectl apply -f secret 
or 
创建secret的另外一种方式：kubectl create secret generic cos-secret -n kube-system  --from-literal=SecretId=AKIDjustfortest --from-literal=SecretKey=justfortest
![cos-secret](/assets/images/2020/07/cosfscos-secret.png) 
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

kubectl apply -f pv.yaml

kubectl apply -f pvc.yaml

      

 
![pv-pvc1](/assets/images/2020/07/cosfs/pv-pvc1.png) 
![pv-pvc](/assets/images/2020/07/cosfs/pv-pvc.png)
##### 3. Create a Pod to use the PVC
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