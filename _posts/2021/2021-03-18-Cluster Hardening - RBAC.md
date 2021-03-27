---
layout: post
title: Cluster Hardening - RBAC
date: 2021-03-18 10:00:00
category: cks
tags:  kubernetes cks RBAC
author: duiniwukenaihe
---
* content
{:toc


注：通过RBAC对集群进行加固
# 关于RBAC 
### RBAC-role based access control  基于角色的访问控制
注：ABAC（基于属性的访问控制），ABAC木有接触过...


# 1. Introduction to RBAC  RBAC介绍


**"Role-based access control (RBAC) is a method of regulating accessto computer or network resources based on the roles of individualusers within your organization."**
**基于角色的访问控制(RBAC)是一种根据个人用户在组织中的角色来管理对计算机或网络资源的访问的方法。**
**`RBAC`使用`rbac.authorization.k8s.io` API Group 来实现授权决策，允许管理员通过 Kubernetes API 动态配置策略，要启用`RBAC`，需要在 apiserver 中添加参数`--authorization-mode=RBAC`，如果使用的`kubeadm`安装的集群，1.6 版本以上的都默认开启了`RBAC`，可以通过查看 Master 节点上 apiserver 的静态`Pod`定义文件：**
![image.png](https://img-blog.csdnimg.cn/img_convert/eac6200052bf43b554e27cf0fc0c9331.png#align=left&display=inline&height=231&margin=[objectObject]&name=image.png&originHeight=462&originWidth=848&size=47716&status=done&style=none&width=424)


![image.png](https://img-blog.csdnimg.cn/img_convert/3c54714364136486f22ed9ce84cd88fe.png#align=left&display=inline&height=223&margin=[objectObject]&name=image.png&originHeight=446&originWidth=877&size=105716&status=done&style=none&width=438.5)


**RBAC  流程定义三大组件：     subject  Role  rolebinding   主体  授权规则 准入控制**
## 1. 访问Kubernetes资源时，限制对Kubernetes资源的访问基于用户或ServiceAccount
## 2. 使用角色和绑定
## 3. 指定允许的内容，其他所有内容都将被拒绝.可以设置白名单
![image.png](https://img-blog.csdnimg.cn/img_convert/7dabb959d5778c2507c7b6adddda68e0.png#align=left&display=inline&height=217&margin=[objectObject]&name=image.png&originHeight=434&originWidth=862&size=67190&status=done&style=none&width=431)
## 4. POLP-Priciple of Least Privilege最小特权原则


            ** 仅访问合法目的所需的数据或信息**
![image.png](https://img-blog.csdnimg.cn/img_convert/415d2f5ad2f14ad16888a537a4eb3a97.png#align=left&display=inline&height=223&margin=[objectObject]&name=image.png&originHeight=445&originWidth=869&size=52756&status=done&style=none&width=434.5)


# 2. RBAC的资源分类  命名空间级别and 非命名空间级别（集群级别）


![image.png](https://img-blog.csdnimg.cn/img_convert/f1e2eebb3dceefa9b50e08179598f036.png#align=left&display=inline&height=534&margin=[objectObject]&name=image.png&originHeight=534&originWidth=832&size=114130&status=done&style=none&width=832)
常用查询命令
```html
查询在namespace中的资源对象
执行命令：kubectl api-resources --namespaced=true
查询不在namespace中的资源对象
执行命令：kubectl api-resources --namespaced=false
查询资源对象与namespace的关系
执行命令：kubectl api-resources
```


![image.png](https://img-blog.csdnimg.cn/img_convert/e8a5b68b6214a5d3d5fec014705844ee.png#align=left&display=inline&height=258&margin=[objectObject]&name=image.png&originHeight=515&originWidth=1059&size=60044&status=done&style=none&width=529.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/a4b97a76fe14a24fa8c0982200d77007.png#align=left&display=inline&height=199&margin=[objectObject]&name=image.png&originHeight=397&originWidth=1245&size=62603&status=done&style=none&width=622.5)
# 3. 关于role rolebinding   clusterrole  clusterrolebinding
其实从上面第五节的:kubectl api-resources --namespaced=true
 kubectl api-resources --namespaced=false两个图中就能看出来：
## 1. role   rolebinding是限制于命名空间的。
## 2. clusterrole和clusterrolebinding是适用于全部命名空间的没有命名空间的限制，适用于整个集群。
### ![image.png](https://img-blog.csdnimg.cn/img_convert/5c1af1efa90093d1b6851db5e614b7bf.png#align=left&display=inline&height=547&margin=[objectObject]&name=image.png&originHeight=547&originWidth=956&size=174206&status=done&style=none&width=956)




## 3. 关于role
相同的角色名称在不同的命名空间中表现不同,如下面的例子blue与red命名空间中都有名为secret-manager的role.
用户x可以是多个命名空间中的秘钥管理员，但是权限可以是不同的。


[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-c7XTf2uH-1616034960454)(https://cdn.nlark.com/yuque/0/2021/png/2505271/1611306105053-09cb1212-ee0c-482a-894e-95b2e2d14579.png#align=left&display=inline&height=532&margin=%5Bobject%20Object%5D&name=image.png&originHeight=532&originWidth=963&size=145704&status=done&style=none&width=963)]
## 4. 关于clusterrole
clusterrole是没有命名空间限制的 权限针对与cluster 集群。
clusterrole在集群中所有的命名空间的都是相同的
用户x可以是多个命名空间中的秘密管理员，每个用户的权限都相同


![image.png](https://img-blog.csdnimg.cn/img_convert/ba376c20993c4f9bfcda3e37639157ec.png#align=left&display=inline&height=247&margin=[object Object]&name=image.png&originHeight=494&originWidth=873&size=109955&status=done&style=none&width=436.5)
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-o3Q9lxE3-1616034960456)(https://cdn.nlark.com/yuque/0/2021/png/2505271/1611306208655-6ef33b31-9265-47e1-819d-ce8525499f64.png#align=left&display=inline&height=536&margin=%5Bobject%20Object%5D&name=image.png&originHeight=536&originWidth=966&size=98813&status=done&style=none&width=966)]


# 3. RBAC实现方式
### 1. Role->RoleBinding  用户具有单个命名空间的权限




![image.png](https://img-blog.csdnimg.cn/img_convert/3c2e3832664e3591d734c972c4eefea2.png#align=left&display=inline&height=245&margin=[object Object]&name=image.png&originHeight=490&originWidth=882&size=108603&status=done&style=none&width=441)
### 2. ClusterRole->ClusterRoleBinding 用户具有全部命名空间的相同权限




![image.png](https://img-blog.csdnimg.cn/img_convert/1e6fade2188b1d3f99ee5e4d1743db00.png#align=left&display=inline&height=528&margin=[object Object]&name=image.png&originHeight=528&originWidth=1074&size=143323&status=done&style=none&width=1074)
### 3. ClusterRole->RoleBinding  用户在多个命名空间中具有相同的权限(**会使clusterrole降级)**
RoleBinding对象也可以引用一个ClusterRole对象用于在RoleBinding所在的命名空间内授予用户对所引用的ClusterRole中 定义的命名空间资源的访问权限。
这一点允许管理员在整个集群范围内首先定义一组通用的角色，然后再在不同的命名空间中复用这些角色.
csdn小凡这里写的有各种的例子[https://zzq23.blog.csdn.net/article/details/109680359](https://zzq23.blog.csdn.net/article/details/109680359)。
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-X4CeMXCY-1616034960458)(https://cdn.nlark.com/yuque/0/2021/png/2505271/1611306343628-6166250a-bdfe-4999-b81b-6aec4dfb99a4.png#align=left&display=inline&height=545&margin=%5Bobject%20Object%5D&name=image.png&originHeight=545&originWidth=1037&size=137253&status=done&style=none&width=1037)]
### 4. role和 Clusterrolebinding是无法绑定的
![image.png](https://img-blog.csdnimg.cn/img_convert/f0e3fe9a91175a88e26154e3c8ae8956.png#align=left&display=inline&height=522&margin=[object Object]&name=image.png&originHeight=522&originWidth=983&size=135779&status=done&style=none&width=983)
### 5. 权限是可以累加的
![image.png](https://img-blog.csdnimg.cn/img_convert/4cb0195c1f4e9742676b2a1f7425b518.png#align=left&display=inline&height=503&margin=[object Object]&name=image.png&originHeight=503&originWidth=1010&size=126373&status=done&style=none&width=1010)
更为详细的还是看官方文档：[https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
# 4.RBAC各种场景
## 1. Role-> RoBinding
![image.png](https://img-blog.csdnimg.cn/img_convert/b519f82e85e9efce1b370738b288d94b.png#align=left&display=inline&height=564&margin=[object Object]&name=image.png&originHeight=564&originWidth=984&size=184006&status=done&style=none&width=984)

- 创建 red blue两个命名空间
- 赋予用户jane在red空间内可以get secrets的权限
- 赋予用户jane在blue空间可以get list secrets的权限



```html
kubectl create ns red
kubectl create ns blue
kubectl -n red create role secret-manager --verb=get --resource=secrets  -oyaml 

kubectl -n red create rolebinding secret-manager --role=secret-manager --user=jane 

kubectl -n blue create role secret-manager --verb=get --verb=list --resource=secrets
kubectl -n blue create rolebinding secret-manager --role=secret-manager --user=jane
# check permissions


```

- 使用auth can-i进行测试

kubectl  red auth can-i -h 获取帮助
```html

root@cks-master:~/rbac# kubectl -n red auth can-i get secrets --as jane
yes
root@cks-master:~/rbac# kubectl -n red auth can-i get secrets --as tom
no
root@cks-master:~/rbac# kubectl -n red auth can-i delete secrets --as tom
no
root@cks-master:~/rbac# kubectl -n red auth can-i list secrets --as tom
no
root@cks-master:~/rbac# kubectl -n blue auth can-i list secrets --as tom
no
root@cks-master:~/rbac# kubectl -n blue auth can-i list secrets --as jane
yes
root@cks-master:~/rbac# kubectl -n blue auth can-i get secrets --as jane
yes
```
![image.png](https://img-blog.csdnimg.cn/img_convert/d1d416b5c021ac3c2ecca1ee072a0067.png#align=left&display=inline&height=312&margin=[object Object]&name=image.png&originHeight=624&originWidth=1495&size=85626&status=done&style=none&width=747.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/5008d2ee5ae719c59f708f1f81acaca1.png#align=left&display=inline&height=119&margin=[object Object]&name=image.png&originHeight=237&originWidth=1071&size=30268&status=done&style=none&width=535.5)
## 2. Clusterrole-> (Cluster)RoleBding
![image.png](https://img-blog.csdnimg.cn/img_convert/739dd3f42bcd4ae5ec37b73ef704cf50.png#align=left&display=inline&height=245&margin=[object Object]&name=image.png&originHeight=489&originWidth=876&size=140179&status=done&style=none&width=438)

- 创建 名为deploy-deleter的 clusterrole 可以删除deployments
- 赋予用户jane可以在所有空间内删除deployments的权限
- 赋予用户jim只能在red空间内删除deployments的权限
- auth can-i测试

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-SwG0bhgC-1616034960461)(https://cdn.nlark.com/yuque/0/2021/png/2505271/1615879995837-b5cec24b-306f-4cfa-a4cd-2246c4be6fd2.png#align=left&display=inline&height=223&margin=%5Bobject%20Object%5D&name=image.png&originHeight=446&originWidth=943&size=60146&status=done&style=none&width=471.5)]
```html
root@cks-master:~/rbac# kubectl create clusterrole deploy-deleter --verb=delete --resource=deployments
dclusterrole.rbac.authorization.k8s.io/deploy-deleter created
root@cks-master:~/rbac# kubectl create clusterrolebinding deploy-deleter --user jane --clusterrole=deploy-deleter
clusterrolebinding.rbac.authorization.k8s.io/deploy-deleter created
root@cks-master:~/rbac# kubectl create rolebinding deploy-deleter --user=jim --clusterrole=deploy-deleter -n red
rolebinding.rbac.authorization.k8s.io/deploy-deleter created

```
```html
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jane
yes
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jane -A
yes
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jane -n default
yes
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jane -n red
yes
root@cks-master:~/rbac# kubectl auth can-i delete pods --as jane -n red
no
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jim -n default
no
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jim -A
no
root@cks-master:~/rbac# kubectl auth can-i delete deployments --as jim -n red
yes

```
![image.png](https://img-blog.csdnimg.cn/img_convert/367ae86df12d92a77b4e8ac2f35b1c68.png#align=left&display=inline&height=213&margin=[object Object]&name=image.png&originHeight=425&originWidth=1291&size=65518&status=done&style=none&width=645.5)
## 3.  Accounts and Users
### 1. serviceaccount and  nornal user
#### 1. kubernetes api管理的serviceaccount资源
#### 2. 普通用户：不是kubernetes 管理的独立于集群服务的管理用户，由谷歌或者亚马逊这样的云平台管理的用户

- 分配私钥的管理员
- 用户商店（例如Keystone或Google帐户）
- 包含用户名和密码列表的文件

详见：[https://kubernetes.io/docs/reference/access-authn-authz/authentication/](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
![image.png](https://img-blog.csdnimg.cn/img_convert/909f141157c213d227be36dd736538f0.png#align=left&display=inline&height=246&margin=[object Object]&name=image.png&originHeight=492&originWidth=866&size=131061&status=done&style=none&width=433)
### 2. 关于用户和证书（这是normal user体系）

- 独立于k8s体系
- 使用证书与秘钥与k8s集群交互

关于证书与集群的交互过程就先不求甚解，后期补上。
![image.png](https://img-blog.csdnimg.cn/img_convert/b142119989d3ca1b12bb92a39b9bae0b.png#align=left&display=inline&height=569&margin=[object Object]&name=image.png&originHeight=569&originWidth=946&size=130479&status=done&style=none&width=946)
#### 1. CSR-CertificateSigningRequest---证书签名请求
关于证书签名请求的过程，还是不求甚解了。证书的有时间单独拿出来研究了。
![image.png](https://img-blog.csdnimg.cn/img_convert/32ce8e612f0789c9dd1ebec174b28951.png#align=left&display=inline&height=242&margin=[object Object]&name=image.png&originHeight=483&originWidth=870&size=90494&status=done&style=none&width=435)
#### 2. Users and Certifiates-leak + Invalidation 用户与证书的  泄漏+无效
无法使证书无效
如果证书泄漏

- 删除通过RBAC的所有访问权限
- 证书过期之前不能使用用户名
- 创建新的CA并重新颁发所有证书



![image.png](https://img-blog.csdnimg.cn/img_convert/0d3fc03c59c5ddadea634c03398bdf80.png#align=left&display=inline&height=247&margin=[object Object]&name=image.png&originHeight=493&originWidth=874&size=93248&status=done&style=none&width=437)


## 5. Users and Certifcates
![image.png](https://img-blog.csdnimg.cn/img_convert/cf3c722879a41fd0f484a743cebd8266.png#align=left&display=inline&height=529&margin=[object Object]&name=image.png&originHeight=529&originWidth=902&size=155857&status=done&style=none&width=902)
### 1. Create Key and Create CSR
```html
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr # only set Common Name = jane
```
![image.png](https://img-blog.csdnimg.cn/img_convert/9c56475d55090d760e0e5947b3dc496b.png#align=left&display=inline&height=234&margin=[object Object]&name=image.png&originHeight=468&originWidth=1118&size=56174&status=done&style=none&width=559)

### 2. 使用kubernetes API签署证书签名请求
![image.png](https://img-blog.csdnimg.cn/img_convert/044c3654f302fe29d7e5c80cbc8577a7.png#align=left&display=inline&height=471&margin=[object Object]&name=image.png&originHeight=942&originWidth=1413&size=176842&status=done&style=none&width=706.5)
[http://blog.itpub.net/69955379/viewspace-2681334/](http://blog.itpub.net/69955379/viewspace-2681334/)  vim删除单行多行
[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-j4Cl5qbS-1616034960466)(https://cdn.nlark.com/yuque/0/2021/png/2505271/1615987449192-408b9050-e721-4c10-bf81-e2b4001fcddb.png#align=left&display=inline&height=153&margin=%5Bobject%20Object%5D&name=image.png&originHeight=305&originWidth=1354&size=68617&status=done&style=none&width=677)]
![image.png](https://img-blog.csdnimg.cn/img_convert/541ff94c331ea6d3835369762e1e93f1.png#align=left&display=inline&height=435&margin=[object Object]&name=image.png&originHeight=869&originWidth=1920&size=233765&status=done&style=none&width=960)


```html
root@cks-master:~/work/key# kubectl create -f csr.yaml
certificatesigningrequest.certificates.k8s.io/jane created
root@cks-master:~/work/key# cat csr.yaml 
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ21UQ0NBWUVDQVFBd1ZERUxNQWtHQTFVRUJoTUNRVlV4RXpBUkJnTlZCQWdNQ2xOdmJXVXRVM1JoZEdVeApJVEFmQmdOVkJBb01HRWx1ZEdWeWJtVjBJRmRwWkdkcGRITWdVSFI1SUV4MFpERU5NQXNHQTFVRUF3d0VhbUZ1ClpUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1BanRQakxIa2JlSlEySzFsM2oKSXJ0dWkwSDY5ZUh6bzQ4c0liMDZsamk2ZlBMS1lDQ2hjbG84ajF1YUlyNHhZUHZEZzBCbnhVNExaVDZFVjErZApDT01teFJOa3ZZaGRIZE9XMm5RSll1RGN3K2M5WXo3NjVLMllnWFE5NFhGaFkvYW9LdWRVM1pzMDhQdkwrenhXClNEbGxBOWorWDJZNExESnV6emNDZDlkRTNQcFlQUXFSQUZFdzVPd3krbHlhRWhKdkZxV29IeDdCSFNWdzcxZW8KTTJZaFhnQ0Q2eDNXY0lVQ0Z6Q1FBWmtkQ1g0SCs5Sk1XVE1xa0hjQWN2Mk8zREpDK25lSGg5S1pKbnJ1RzNvTwpIc1BvcFZsZzBiZ1V3TWFvSlg2K1dpWW5EOWszbVRIenlhTjVjQ1l5YkZNa0lkZmZXTE5kWTJTU2hMQThMRzFUCk16c0NBd0VBQWFBQU1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ21UbDBXejNQOWhpY2xJcnhPTTB3LzFXMFoKQWtRYlYvNjU5MU8rOUNUNVJFYnFFbUxhY2RwazQ3NGpKR1ZvdjREbVJvdVlPbXNPWXFtem5acWxhL0MwMGMwRQpRUkxWWjJpNGlLWTNpSWpjSWdoVWQzWUZHbk84NEhNU3NUNEIzaDMxZWtTNFF1YVRDSnFGYU84eHNDeTdpMElnClRYcnlJNCtaOEhoeElVcGNZZ0xaRDR5K3BNYUxZSnVxSmtuUU1idy9QbU5ra0trOUEwWlp6Ym9aRVNUelhpeWEKcVB3azlxZXladzJSOGo0cVI3aXlMQ0VZL05DdkFBQXNzN28zYkVSV1ltbTQveHRPT0NFN2lKRk15MWNmczVJbwpPNnptNndaYUl6dDVUNmh5TUp2RWdCV3BMd3R5Nmt1TFlNRzhNR0lnakJJY1p4ckh2Vm9xWXJ2VXk3ejMKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
root@cks-master:~/work/key# kubectl get csr
NAME   AGE   SIGNERNAME                            REQUESTOR          CONDITION
jane   21s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Pending
root@cks-master:~/work/key# kubectl certificate -h
Modify certificate resources.

Available Commands:
  approve     Approve a certificate signing request
  deny        Deny a certificate signing request

Usage:
  kubectl certificate SUBCOMMAND [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
root@cks-master:~/work/key# kubectl certificate approve jane
certificatesigningrequest.certificates.k8s.io/jane approved
root@cks-master:~/work/key# kubectl get csr
NAME   AGE    SIGNERNAME                            REQUESTOR          CONDITION
jane   119s   kubernetes.io/kube-apiserver-client   kubernetes-admin   Approved,Issued

```
![image.png](https://img-blog.csdnimg.cn/img_convert/11c9d82ada8b496de72aaedca5ae32b0.png#align=left&display=inline&height=301&margin=[object Object]&name=image.png&originHeight=602&originWidth=1919&size=123851&status=done&style=none&width=959.5)
kubectl get csr jane -o yaml
![image.png](https://img-blog.csdnimg.cn/img_convert/89a0130ca494a0908a279e579f2f688b.png#align=left&display=inline&height=432&margin=[object Object]&name=image.png&originHeight=863&originWidth=1910&size=186604&status=done&style=none&width=955)
![image.png](https://img-blog.csdnimg.cn/img_convert/0f1c1e45baaa11ab28f58ea6cb311721.png#align=left&display=inline&height=328&margin=[object Object]&name=image.png&originHeight=655&originWidth=1920&size=220906&status=done&style=none&width=960)
![image.png](https://img-blog.csdnimg.cn/img_convert/c3c4f0fe0318ee654939b08943991b66.png#align=left&display=inline&height=45&margin=[object Object]&name=image.png&originHeight=90&originWidth=914&size=9694&status=done&style=none&width=457)
### 3. Cert+Key签署CSR连接K8sAPI
```html
kubectl config set-credentials jane --client-key=jane.key --client-certificate=jane.crt
kubectl config set-credentials jane --client-key=jane.key --client-certificate=jane.crt --embed-certs
```


![image.png](https://img-blog.csdnimg.cn/img_convert/9a18d58eb1d388dd1e37bfd208aa4b38.png#align=left&display=inline&height=398&margin=[object Object]&name=image.png&originHeight=796&originWidth=986&size=74538&status=done&style=none&width=493)
设置上下文参数,包含集群名称和访问集群的用户名字 
```html
kubectl config set-context jane --cluster=kubernetes --user=jane
```
![image.png](https://img-blog.csdnimg.cn/img_convert/df2f4e7be024240de5b0e2c9f6cbdbe7.png#align=left&display=inline&height=275&margin=[object Object]&name=image.png&originHeight=550&originWidth=953&size=51235&status=done&style=none&width=476.5)
切换jane为默认用户 auth can-i测试
```html
root@cks-master:~/work/key# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          jane                          kubernetes   jane               
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
root@cks-master:~/work/key# kubectl config u
unset        use-context  
root@cks-master:~/work/key# kubectl config use-context jane
Switched to context "jane".
root@cks-master:~/work/key# kubectl get ns
Error from server (Forbidden): namespaces is forbidden: User "jane" cannot list resource "namespaces" in API group "" at the cluster scope
root@cks-master:~/work/key# kubectl get secrets -n blue
NAME                  TYPE                                  DATA   AGE
default-token-hb47b   kubernetes.io/service-account-token   3      34h
root@cks-master:~/work/key# kubectl delete secret default-token-hb47b -n blue
Error from server (Forbidden): secrets "default-token-hb47b" is forbidden: User "jane" cannot delete resource "secrets" in API group "" in the namespace "blue"
root@cks-master:~/work/key# kubectl auth can-i delete deployment -A
yes
root@cks-master:~/work/key# kubectl auth can-i delete pods -A
no

```
![image.png](https://img-blog.csdnimg.cn/img_convert/8a1bcda59b58a352f9e1641c2bde5769.png#align=left&display=inline&height=110&margin=[object Object]&name=image.png&originHeight=220&originWidth=1601&size=37963&status=done&style=none&width=800.5)
















