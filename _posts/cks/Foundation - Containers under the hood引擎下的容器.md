---
layout: "post"
title: "Foundation - Containers under the hood引擎下的容器"
date: "2021-03-13 18:00:00"
category: "kubernetes cks"
tags:  "Foundation - Containers under the hood引擎下的容器"
author: duiniwukenaihe
---
* content
{:toc}



## 1. Containers  and Images


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612249687928-723e13b1-4f36-44e1-98cf-5a1c5847ac75.png#align=left&display=inline&height=706&margin=%5Bobject%20Object%5D&name=image.png&originHeight=706&originWidth=1372&size=300420&status=done&style=none&width=1372)


1. **Dockerfile- Script/text defined how to build an image  脚本或者文本 定义了如何去创建一个镜像**
1. **Image-通过docker build构建的多层的二进制表现形式**
1. **通过docker run  可以运行该镜像的实例**
1. **也可以通过docker push将镜像上传到镜像仓库，然后通过docker pull 将镜像下载到本地服务器 然后运行一个容器实例**
## 2. 关于Container容器
**![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612250288390-9939ff54-2303-4983-9cc5-020b8e625102.png#align=left&display=inline&height=700&margin=%5Bobject%20Object%5D&name=image.png&originHeight=700&originWidth=1313&size=165846&status=done&style=none&width=1313)**

1. **  Collection  of one or multiple applications-收集一个或多个应用程序**
1. **  Includes all its dependencies-包括所有的依赖项**
1. **  Just process which runs on the Linux Kernel(but wich cannot see everything)-只是在Linux内核上运行的进程（但是无法看到所有内容）**
## 3. Kernel vs User Space  内核vs 用户命名空间
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612250612814-57955997-d48c-4408-9b0f-220584a4cdf6.png#align=left&display=inline&height=780&margin=%5Bobject%20Object%5D&name=image.png&originHeight=780&originWidth=1385&size=340651&status=done&style=none&width=1385)
**容器和系统调用**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612251437717-8fd6c5cf-e928-4f89-8c24-60ed9e563ccf.png#align=left&display=inline&height=660&margin=%5Bobject%20Object%5D&name=image.png&originHeight=660&originWidth=1038&size=142597&status=done&style=none&width=1038)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612252287464-bedc75fa-9644-4a09-b323-3538c26ec6aa.png#align=left&display=inline&height=669&margin=%5Bobject%20Object%5D&name=image.png&originHeight=669&originWidth=1092&size=176383&status=done&style=none&width=1092)
这个图就是为了突出 container 都运行在kernel层上面
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612252359539-515dbac5-efe1-4155-8007-b5427963d58e.png#align=left&display=inline&height=691&margin=%5Bobject%20Object%5D&name=image.png&originHeight=691&originWidth=1231&size=284104&status=done&style=none&width=1231)
是不是可以理解为containers中app是直接运行在宿主机 的内核之上， vm虚拟机中的app是运行在虚拟机操作系统的内核上而不是宿主机的内核之上？


## 4. Linux  kernel namespace
### PID 

1. **isolates processes from each other  进程相互隔离**
1. ** one process cannot see others  一个进程看不到其他进程**
1. **process id 10 can exist multiple time,onece in every namespace  进程ID 10可以存在多个时间，每个命名空间只能出现一次**
### Mount

1. **restrict  access to mounts or root filesystem    限制对挂载或根文件系统的访问**
### Network

1. **only access certain network devices  仅访问某些确定的网络设备**
1. **firewall& routing rules& socket port numbers  防火墙&路由规则&套接字端口号**
1. **not able to see all traffic or contact all endpoints 无法查看所有流量或无法联系所有端点**
### User

1. **different set of user ids used  使用了不同的用户ID集**
1. **user(0)inside one namepsace can be different from user(0)inside another  命名空间内的用户（0）可以不同于另一个空间内的用户（0）**
1. **dont use the host-root(0)inside a countainer  不要在容器内使用主机根（0）**

**
## 5. 关于namespace和cgroup的隔离
**
**cgroups 限制进程使用的资源**
**Ram**
**Disk**
**CPU**
**
**namespaces 限制可以看到的进程**
**其他进程**
**用户**
**文件系统**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612253074156-ed13a141-e0bb-4d3d-aebd-aba41a07cddd.png#align=left&display=inline&height=683&margin=%5Bobject%20Object%5D&name=image.png&originHeight=683&originWidth=1228&size=164774&status=done&style=none&width=1228)


## 6. Docker isolation in action 进行docker隔离的一个例子
**例子：创建两个容器并检查它们是否彼此看不见**
**         两个容器运行与相同命名空间**

```
root@cks-master:~# docker run --name c1 -d ubuntu sh -c "sleep 1d"
bc244ea97b2c0053dfa3df81580973034683fdda0c56abafe4ba4705b22866be
root@cks-master:~# docker exec c1 ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.2  0.0   2612   532 ?        Ss   09:25   0:00 sh -c sleep 1d
root         6  0.0  0.0   2512   588 ?        S    09:25   0:00 sleep 1d
root         7  0.0  0.0   5900  2960 ?        Rs   09:25   0:00 ps aux
root@cks-master:~# docker run --name c2 -d ubuntu sh -c "sleep 999d"
44393c4c47b07d6b210acdd30440d5f4c8d8fa836f9aa4b1eb51e53556c03ccf
root@cks-master:~# docker exec c2 ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.1  0.0   2612   536 ?        Ss   09:25   0:00 sh -c sleep 999d
root         6  0.0  0.0   2512   596 ?        S    09:25   0:00 sleep 999d
root         7  0.0  0.0   5900  2852 ?        Rs   09:25   0:00 ps aux
root@cks-master:~# ps -aux|grep sleep
root      6897  0.0  0.0   2612   532 ?        Ss   17:25   0:00 sh -c sleep 1d
root      6939  0.0  0.0   2512   588 ?        S    17:25   0:00 sleep 1d
root      7148  0.0  0.0   2612   536 ?        Ss   17:25   0:00 sh -c sleep 999d
root      7186  0.0  0.0   2512   596 ?        S    17:25   0:00 sleep 999d
root      7405  0.0  0.0  13780  1016 pts/0    S+   17:26   0:00 grep --color=auto sleep
root@cks-master:~# docker rm c2 --force
c2
root@cks-master:~# docker run --name c2 --pid=contariner:c1 -d ubuntu sh -c "sleep 999d"
docker: --pid: invalid PID mode.
See 'docker run --help'.
root@cks-master:~# docker run --name c2 --pid=container:c1 -d ubuntu sh -c "sleep 999d"
31a8e174129c86387d3f696d3b6166a860fa2f1ac878efca81c5f9a6d68d5486
root@cks-master:~# docker exec c2 ps -aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2612   532 ?        Ss   09:25   0:00 sh -c sleep 1d
root         6  0.0  0.0   2512   588 ?        S    09:25   0:00 sleep 1d
root        13  0.0  0.0   2612   608 ?        Ss   09:26   0:00 sh -c sleep 999d
root        18  0.0  0.0   2512   596 ?        S    09:26   0:00 sleep 999d
root        19  0.0  0.0   5900  2820 ?        Rs   09:33   0:00 ps -aux
root@cks-master:~# docker exec c1 ps -aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2612   532 ?        Ss   09:25   0:00 sh -c sleep 1d
root         6  0.0  0.0   2512   588 ?        S    09:25   0:00 sleep 1d
root        13  0.0  0.0   2612   608 ?        Ss   09:26   0:00 sh -c sleep 999d
root        18  0.0  0.0   2512   596 ?        S    09:26   0:00 sleep 999d
root        24  0.0  0.0   5900  2988 ?        Rs   09:34   0:00 ps -aux

```


