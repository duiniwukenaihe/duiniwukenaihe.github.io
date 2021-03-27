---
layout: post
title: K8s Security Best Practices-K8S安全最佳实践
date: 2021-01-28 14:00:00
category: cks
tags:  kubernetes cks
author: duiniwukenaihe
---
* content
{:toc}

 1. **关于安全-写在前面的：**
  **Security is complex and a process   安全是复杂的，而且是一个过程**
    1. **Security combines many diffenrent things            安全结合了许多不同的东西**
    2. **Environments change,security cannot stay in a certain state   环境变化，安全性不能保持一定状态**
    3. **Attackers have advantage                                     攻击者有优势**
            3.1 . **They decide time                                 他们决定的时间（任意时间）**
            3.2.  **They pick  what to attack ,like weakest link     他们选择要攻击的内容，例如最薄弱的环节**

## 1. Security Principles 安全原则

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191033127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)

1. **Defense in depth                 深度防御**
2. **Least Privilege                     最小权限原则**
3. **Limiting the Attack Surface  限制攻击面**
4. **Layered defence and Redundancy    分层防御和冗余**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191101433.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)


## 2. K8s Security Categories k8S安全分类

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191112204.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)


### 2.1 Host OS Security 宿主机操作系统安全

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191119399.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)
```bash

 1. kubernets Node should only do one thing :  Kubernetes 
    kubernets节点应该只运行kubernets
    1.1. Reduce Attack Surface   减少攻击面             
    1.2. Remove unnecessary applications   移除不必要的应用                          
 2. Keep up to date     保持升级
 3. Rutime security tools     运行时安全工具
 4. Find and identify malicious processes  查找和识别恶意进程
 5. Restrict IAM  ssh access      限制IAM(Identity and Access Management  身份识别与访问管理服务) 与SSH访问

```

### 2.2 Kubernetes Cluster Security k8s集群安全

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191133571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)
**kubernetes  components are runing secure and up-to-data      kubernetes集群组件保持安全和最新的运行**

 **1. apiserver
 2.  kubelet
 3.  etcd**

**Restrict(external)access                                                               限制（外部）访问
use  authentication ->authorization                                           使用身份认证 授权
Admission Controllers                                                                控制器准入？
  noderestriction                                                                  节点限制
  custom policies(opa)                                                         自定义规则
Enable Audit Logging                                                               启用审计日志记录
Security Benchmarking                                                             安全基线测试**

由此可见最新的还是很重要的，老的版本会有漏洞，有必要保证组件版本的更新升级
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191143875.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)
一个通过 pod 攻击etcd的例子展现与etcd的安全策略：
  **加密etcd
  限制访问etcd
  加密与etcd的通信**

### 2.3 Application Security 应用安全

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210310191149839.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NheW5haWhl,size_16,color_FFFFFF,t_70)

```bash
 1. Use Secrets /no hardcoded credentials  使用secrets秘钥 而不是硬编码的凭据

 2. RBAC                       基于角色的访问控制
 
 3. Container Sandboxing                          容器沙盒

 4. Container Hardening                            容器加固

   4.1 Attack Surface                             攻击表面
   
   4.2 Run as user                                 作为用户运行 no root
   
   4.3 Readonly filesystem                    只读的文件系统
   
5. Vulnerability Scanning                         漏洞扫描

6. MTLS/ServiceMeshes                          双向认证/服务网格在这里插入代码片
```



