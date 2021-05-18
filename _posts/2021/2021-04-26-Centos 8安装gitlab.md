---
layout: post
title: Centos 8安装gitlab
date: 2021-04-26 6:00:00
category: Centos
tags: centos gitlab
author: duiniwukenaihe
---


# 背景：

本来吧gitlab都是搞在kubernetes上面的。公司内网环境需要一个内部的gitlab.然后就准备搭建一个。另外跟着阳明大佬的课程做gitlab触发jenkins任务的时候我的jenkins想拿内网的去做，没法去触发啊。正好搞一个内网的去玩一下呢。线上的都是on kubernetes 的不想太随意去各种玩了......

# 一. centos8安装gitlab过程

## 1. 下载rpm包

很多可以下载的。另外gitlab成立了中国的独立的公司极狐？也可以玩一下。不过最近的极狐都被华为造车的的极狐的风头盖过去了吧......

下载源选择了清华的源下载了当前最新的版本gitlab-ce-13.9.6-ce.0.el8.x86\_64.rpm

```
wget  https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el8/gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```

![image.png](/assets/images/2021/04-26/vh3odmz5v9.png)

## 2. 安装rpm包

```
rpm -ivh gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```

![image.png](/assets/images/2021/04-26/t00kv4zvgp.png)

显示缺少依赖

```
yum install policycoreutils-python-utils
rpm -ivh gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```

![image.png](/assets/images/2021/04-26/4yzr4p1ipk.png)

## 3. 修改gitlab配置文件

gitlab的配置文件是`/etc/gitlab/gitlab.rb`。我个人是修改了域名还有存储位置（系统盘太小了。挂载了数据盘）

修改/etc/gitlab/gitlab.rb文件：

```
external_url 'http://192.168.0.173'
git_data_dirs({ "default" => { "path" => "/data/gitlab" } })
```

![image.png](/assets/images/2021/04-26/us4btf8p4h.png)

![image.png](/assets/images/2021/04-26/j3v8evijy3.png)

当然了文件夹我已经建好了：

```
mkdir  /data/gitlab
```

加载配置：

```
 gitlab-ctlreconfigure
```

![image.png](/assets/images/2021/04-26/plo2ukfmbl.png)

关于服务的运行控制：

```
## 启动服务
gitlab-ctl start
## 重启服务
gitlab-ctl restart 
## 查看状态
gitlab-ctl status 
## 停止
gitlab-ctl stop
```

注意：external\_url貌似没有那么重要.....我开始吧ip输入错了 仍能正常访问。没有那么严格校验的

1. 访问web测试

浏览器访问http://192.168.0.173

![image.png](/assets/images/2021/04-26/prufja2kuo.png)

嗯 账号密码要求大于8位

![image.png](/assets/images/2021/04-26/ltx7ix8f7e.png)

# 二. 搭建下postgresql

正常的gitlab on kubernetes 都搭建了postgresql。分离下搭建个postgresql用一下。参照[https://blog.csdn.net/weixin\_43431593/article/details/106252231](https://blog.csdn.net/weixin_43431593/article/details/106252231)进行了搭建。但是后面权限什么的 吃不下。postgresql自己没有系统玩过。放弃了。选择了docker的安装方式.....毕竟docker的可以偷懒一下......

## 1. centos8安装docker

可参照：[https://blog.csdn.net/qq\_41570843/article/details/106182073](https://blog.csdn.net/qq_41570843/article/details/106182073)。基本都是大同小异。设置阿里云仓库配置加速器。启动docker服务。不做复杂描述：

```
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

这个地方应该会报错的

![image.png](/assets/images/2021/04-26/2csmt2hank.png)

找不到 yum-config-manager?嗯安装一下

```
yum -y install yum-utils
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 yum install docker-ce docker-ce-cli containerd.io -y
```

当然了 也可以配置下镜像加速器：

 cat /etc/docker/daemon.json

```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
      "https://fz5yth0r.mirror.aliyuncs.com",
      "http://hub-mirror.c.163.com/",
      "https://docker.mirrors.ustc.edu.cn/",
      "https://registry.docker-cn.com"
  ],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

```
systemctl daemon-reload
systemctl restart docker
```

## 2. docker搭建postgresql

postgresql选择了了11版本。住：支持是到gitlab14.0。故其实也可以安装更高版本的postgresql。

![image.png](/assets/images/2021/04-26/43v3ziaytr.png)

```
mkdir /data/pgsql

docker run --name dockerPG11 \
-e POSTGRES_PASSWORD=postgres \
-v /data/pgsql:/var/lib/postgresql/data \
-p 54322:5432 \
-d postgres:11.5

## 创建数据库
psql -U postgres -h localhost -p 54322
psql (11.5 (Debian 11.5-3.pgdg90+1))
Type "help" for help.
postgres=# create role gitlab login encrypted password 'gitlab';
CREATE ROLE
postgres=# create database gitlabhq_production owner=gitlab ENCODING = 'UTF8';
CREATE DATABASE
postgres=# \c gitlabhq_production
You are now connected to database "gitlabhq_production" as user "postgres".
gitlabhq_production=# CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION
gitlabhq_production=# CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION
postgres=# \q
```

![image.png](/assets/images/2021/04-26/993r78wr5f.png)

# 三. gitlab与postgresql集成

## 1. 编辑`/etc/gitlab/gitlab.rb`

```
gitlab_rails['db_adapter'] = "postgresql"
gitlab_rails['db_encoding'] = "utf8"
# gitlab_rails['db_collation'] = nil
gitlab_rails['db_database'] = "gitlabhq_production"
gitlab_rails['db_username'] = "gitlab"
gitlab_rails['db_password'] = "gitlab"
gitlab_rails['db_host'] = "127.0.0.1"
gitlab_rails['db_port'] = 54322
postgresql['enable'] = false
```

![image.png](/assets/images/2021/04-26/7tu44j6q6h.png)

![image.png](/assets/images/2021/04-26/k6sa0w5pgo.png)

重载配置文件：

```
gitlab-ctl reconfigure
```

![image.png](/assets/images/2021/04-26/c5yljiokj5.png)

## 2. 验证配置生效

```
cat /opt/gitlab/embedded/service/gitlab-rails/config/database.yml
# This file is managed by gitlab-ctl. Manual changes will be
# erased! To change the contents below, edit /etc/gitlab/gitlab.rb
# and run `sudo gitlab-ctl reconfigure`.

production:
  adapter: postgresql
  encoding: utf8
  collation: 
  database: gitlabhq_production
  username: "gitlab"
  password: "gitlab"
  host: "127.0.0.1"
  port: 54322
  socket: 
  sslmode: 
  sslcompression: 0
  sslrootcert: 
  sslca: 
  load_balancing: {"hosts":[]}
  prepared_statements: false
  statement_limit: 1000
  connect_timeout: 
  keepalives: 
  keepalives_idle: 
  keepalives_interval: 
  keepalives_count: 
  tcp_user_timeout: 
  application_name: 
  variables:
    statement_timeout:
```

继续登陆gitlab页面

![image.png](/assets/images/2021/04-26/tmkah4qn5j.png)

修改默认语言为中文：

![image.png](/assets/images/2021/04-26/2jgw9cn13j.png)

ok。基本算是完成了。这里要特别说一下postgresql的安装登陆个人真的是没有太搞明白。有时间要好好学习下postgresql。和正常自己了解的mysql等数据库比是完全的知识盲区了。有盲区，那就有时间学习一下了。加油......
