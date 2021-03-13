---
layout: "post"
title: "K8s Security Best Practices-K8S安全最佳实践"
date: "2021-03-13 18:00:00"
category: "kubernetes cks"
tags:  "K8s Security Best Practices-K8S安全最佳实践"
author: duiniwukenaihe
---
* content
{:toc}

# 关于安全-写在前面的


**Security is complex and a process   安全是复杂的，而且是一个过程**

**1. Security combines many diffenrent things            安全结合了许多不同的东西**
**2. Environments change,security cannot stay in a certain state   环境变化，安全性不能保持一定状态**
**3. Attackers have advantage                                     攻击者有优势**
**        3.1 . They decide time                                 他们决定的时间（任意时间）**
**        3.2.  They pick  what to attack ,like weakest link     他们选择要攻击的内容，例如最薄弱的环节**

# 1. Security Principles   安全原则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611817513103-6e4355e2-5a71-443f-a4a6-641f2c6c4176.png#align=left&display=inline&height=557&margin=%5Bobject%20Object%5D&name=image.png&originHeight=557&originWidth=875&size=52037&status=done&style=none&width=875)
### 1. Defense in depth                 深度防御
### 2. Least Privilege                     最小权限原则
### 3. Limiting the Attack Surface  限制攻击面
### 4. Layered defence and Redundancy    分层防御和冗余
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611817728231-bf697634-e759-44fd-ba60-e86804f7fc3e.png#align=left&display=inline&height=619&margin=%5Bobject%20Object%5D&name=image.png&originHeight=619&originWidth=932&size=127458&status=done&style=none&width=932)
# 2. K8s Security Categories   k8S安全分类
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611818006133-1f4c9b46-cb91-47ef-87b6-921a3bf8d69e.png#align=left&display=inline&height=551&margin=%5Bobject%20Object%5D&name=image.png&originHeight=551&originWidth=930&size=99937&status=done&style=none&width=930)
## 2.1  Host  OS Security      宿主机操作系统安全
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611818204788-4d22fe09-80a0-42f2-baa9-4c6a79d07c7c.png#align=left&display=inline&height=530&margin=%5Bobject%20Object%5D&name=image.png&originHeight=530&originWidth=902&size=89045&status=done&style=none&width=902)
**kubernets Node should only do one thing :Kubernetes  kubernets节点应该只运行kubernets**
**Reduce Attack Surface                                                     减少攻击面**

- **Remove unnecessary applications                             移除不必要的应用**
- **Keep up to date                                                         保持升级**

**Rutime security tools                                                        运行时安全工具**
**Find and identify malicious processes                              查找和识别恶意进程**
**Restrict IAM  ssh access                                                   限制IAM(Identity and Access Management  身份识别与访问管理服务) 与SSH访问**

## 2.2   Kubernetes  Cluster Security   k8s集群安全
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611819130174-5b219bc9-5ca4-4208-a9c5-63304d7ad5da.png#align=left&display=inline&height=571&margin=%5Bobject%20Object%5D&name=image.png&originHeight=571&originWidth=889&size=108896&status=done&style=none&width=889)






**kubernetes  components are runing secure and up-to-data      kubernetes集群组件保持安全和最新的运行**

- **apiserver**
- **kubelet**
- **etcd**

**Restrict(external)access                                                               限制（外部）访问**
**use  authentication ->authorization                                           使用身份认证 授权**
**Admission Controllers                                                                控制器准入？**

- **  noderestriction                                                                  节点限制**
- **  custom policies(opa)                                                         自定义规则**

**Enable Audit Logging                                                               启用审计日志记录**
**Security Benchmarking                                                             安全基线测试**

由此可见最新的还是很重要的，老的版本会有漏洞，有必要保证组件版本的更新升级
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611820017735-a7d901d8-d347-406b-9952-df8a026e77d0.png#align=left&display=inline&height=535&margin=%5Bobject%20Object%5D&name=image.png&originHeight=535&originWidth=918&size=130291&status=done&style=none&width=918)
一个通过 pod 攻击etcd的例子展现与etcd的安全策略：
**  加密etcd**
**  限制访问etcd**
**  加密与etcd的通信**



## 2.3  Application Security  应用安全
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611820425408-d8e186c9-1700-4b78-a452-36d6db7ca82c.png#align=left&display=inline&height=532&margin=%5Bobject%20Object%5D&name=image.png&originHeight=532&originWidth=957&size=85873&status=done&style=none&width=957)


**Use Secrets /no hardcoded credentials  使用secrets秘钥 而不是硬编码的凭据**
**RBAC    **
**Container Sandboxing                          容器沙盒**
**Container Hardening                            容器加固**

- **   Attack Surface                             攻击表面**
- **   Run as user                                 作为用户运行 no root**
- **   Readonly filesystem                    只读的文件系统**

**Vulnerability Scanning                         漏洞扫描**
**MTLS/ServiceMeshes                          双向认证/服务网格**

