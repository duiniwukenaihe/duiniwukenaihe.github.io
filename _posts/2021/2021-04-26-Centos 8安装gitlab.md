---
layout: post
title: Centos 8安装gitlab
date: 2021-04-26 6:00:00
category: Centos
tags: centos gitlab
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
本来吧gitlab都是搞在kubernetes上面的。公司内网环境需要一个内部的gitlab.然后就准备搭建一个。另外跟着阳明大佬的课程做gitlab触发jenkins任务的时候我的jenkins想拿内网的去做，没法去触发啊。正好搞一个内网的去玩一下呢。线上的都是on kubernetes 的不想太随意去各种玩了......
# 一. centos8安装gitlab过程
## 1. 下载rpm包
很多可以下载的。另外gitlab成立了中国的独立的公司极狐？也可以玩一下。不过最近的极狐都被华为造车的的极狐改过去了吧......
下载源选择了清华的源下载了当前最新的版本gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```
wget  https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el8/gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619430455042-54d01471-1c16-4511-b6da-63147324c746.png#clientId=u29efec65-6235-4&from=paste&height=825&id=uf22f1468&margin=%5Bobject%20Object%5D&name=image.png&originHeight=825&originWidth=1707&originalType=binary&size=117628&status=done&style=none&taskId=u554f4adc-e7e8-420b-9b02-86ba19a8cb1&width=1707)
## 2. 安装rpm包
```
rpm -ivh gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619430568738-f620c8f2-7c4c-4368-96f7-adc1b71c37f7.png#clientId=u29efec65-6235-4&from=paste&height=601&id=ub5116671&margin=%5Bobject%20Object%5D&name=image.png&originHeight=601&originWidth=1243&originalType=binary&size=69100&status=done&style=none&taskId=udf041762-5506-43b1-849a-11da3034619&width=1243)
显示缺少依赖
```
yum install policycoreutils-python-utils
rpm -ivh gitlab-ce-13.9.6-ce.0.el8.x86_64.rpm
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619430695745-efc691b8-9314-4aa2-b2ea-0b434a217731.png#clientId=u29efec65-6235-4&from=paste&height=667&id=u4868f33f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=667&originWidth=1410&originalType=binary&size=67003&status=done&style=none&taskId=u38552af0-daab-4f76-9d85-54b5d80cf65&width=1410)
## 3. 修改gitlab配置文件
gitlab的配置文件是`/etc/gitlab/gitlab.rb`。我个人是修改了域名还有存储位置（系统盘太小了。挂载了数据盘）
修改/etc/gitlab/gitlab.rb文件：
```
external_url 'http://192.168.0.173'
git_data_dirs({ "default" => { "path" => "/data/gitlab" } })

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619430952937-d74d25a2-b0ae-4474-b5a5-7489aea724da.png#clientId=u29efec65-6235-4&from=paste&height=387&id=u5ed1baef&margin=%5Bobject%20Object%5D&name=image.png&originHeight=387&originWidth=807&originalType=binary&size=49958&status=done&style=none&taskId=ua944ac1c-1902-4838-9f4a-50c31bf0f71&width=807)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619430976411-7dc50c63-9d31-4b7b-a729-83bf4d1cbca2.png#clientId=u29efec65-6235-4&from=paste&height=291&id=u593a984e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=291&originWidth=951&originalType=binary&size=40103&status=done&style=none&taskId=u45176df1-96c2-4c51-b497-cb266913e91&width=951)
当然了文件夹我已经建好了：
```
mkdir  /data/gitlab
```
加载配置：
```
 gitlab-ctlreconfigure
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619431484662-4f61b6d4-7347-41ca-b68c-50392a246304.png#clientId=u29efec65-6235-4&from=paste&height=741&id=u108a09d1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=741&originWidth=1171&originalType=binary&size=87080&status=done&style=none&taskId=ueceddf15-dadf-4e08-8bff-2e7b55b0b3b&width=1171)
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
注意：external_url貌似没有那么重要.....我开始吧ip输入错了 仍能正常访问。没有那么严格校验的

4. 访问web测试

浏览器访问http://192.168.0.173
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619431492719-dc53364f-cd86-4ee5-b0af-1cfcffbdc8c0.png#clientId=u29efec65-6235-4&from=paste&height=883&id=uba88eeb4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=883&originWidth=1652&originalType=binary&size=77237&status=done&style=none&taskId=u83497865-564c-4135-bf12-ed3da17c036&width=1652)
嗯 账号密码要求大于8位
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619431522336-f2b4db15-348b-4e9c-8403-8711e62df5f3.png#clientId=u29efec65-6235-4&from=paste&height=690&id=u21da12fd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=690&originWidth=1471&originalType=binary&size=58018&status=done&style=none&taskId=udd5db4f9-9313-482b-929a-57875b1b6ed&width=1471)
# 二. 搭建下postgresql
正常的gitlab on kubernetes 都搭建了postgresql。分离下搭建个postgresql用一下。参照[https://blog.csdn.net/weixin_43431593/article/details/106252231](https://blog.csdn.net/weixin_43431593/article/details/106252231)进行了搭建。但是后面权限什么的 吃不下。postgresql自己没有系统玩过。放弃了。选择了docker的安装方式.....毕竟docker的可以偷懒一下......
## 1. centos8安装docker
可参照：[https://blog.csdn.net/qq_41570843/article/details/106182073](https://blog.csdn.net/qq_41570843/article/details/106182073)。基本都是大同小异。设置阿里云仓库配置加速器。启动docker服务。不做复杂描述：
```
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
这个地方应该会报错的
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619432030057-e0f096f7-9976-49bd-97d2-4903ef2b414a.png#clientId=u29efec65-6235-4&from=paste&height=262&id=u559be035&margin=%5Bobject%20Object%5D&name=image.png&originHeight=262&originWidth=820&originalType=binary&size=32415&status=done&style=none&taskId=u8174f6dc-06b3-40a7-8248-88936417194&width=820)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619435835279-712bd067-e75b-4995-be08-df81c85e541e.png#clientId=u29efec65-6235-4&from=paste&height=143&id=u7fbdf27f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=143&originWidth=1022&originalType=binary&size=14900&status=done&style=none&taskId=u725768e4-ceaa-4974-9b29-85fb0e2ab77&width=1022)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619435796399-ae54b09b-025d-4463-9a58-e59297f5201a.png#clientId=u29efec65-6235-4&from=paste&height=809&id=uad5a52b1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=809&originWidth=1221&originalType=binary&size=111622&status=done&style=none&taskId=u76d95b73-8471-4471-8ff9-40ce03adb4b&width=1221)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619435966082-03d79c1f-7d18-4033-85dd-fa320ccafe1b.png#clientId=u29efec65-6235-4&from=paste&height=341&id=ua45a803e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=341&originWidth=1092&originalType=binary&size=42103&status=done&style=none&taskId=u63d68afe-5298-4082-8be8-89ada661706&width=1092)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619436046783-7edde208-1758-4c7e-a7fb-a3928dc1fa6d.png#clientId=u29efec65-6235-4&from=paste&height=248&id=udc2dc223&margin=%5Bobject%20Object%5D&name=image.png&originHeight=248&originWidth=1023&originalType=binary&size=27441&status=done&style=none&taskId=ud7cad2b0-b00b-4687-8f6e-9b77406f471&width=1023)
重载配置文件：
```
gitlab-ctl reconfigure
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619436129687-d841b79c-cc0b-4fee-9db4-57b2391f682c.png#clientId=u29efec65-6235-4&from=paste&height=547&id=u53669376&margin=%5Bobject%20Object%5D&name=image.png&originHeight=547&originWidth=1057&originalType=binary&size=62677&status=done&style=none&taskId=u1623ac72-4eeb-4f08-8839-de6ac37fbbc&width=1057)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619436316842-f162944d-7b35-49ea-a9de-9e5d8ea0b1e3.png#clientId=u29efec65-6235-4&from=paste&height=770&id=u04bd0ed6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=770&originWidth=1448&originalType=binary&size=65917&status=done&style=none&taskId=u2ee13b91-843f-48c5-8cd2-a567c2dd728&width=1448)
修改默认语言为中文：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1619436326872-b6f9bb2b-dbb7-4204-b695-26ea6290cd24.png#clientId=u29efec65-6235-4&from=paste&height=970&id=uaecedda3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=970&originWidth=1706&originalType=binary&size=132918&status=done&style=none&taskId=u1a7fc5da-5490-46a5-89a3-23915104721&width=1706)
ok。基本算是完成了。这里要特别说一下postgresql的安装登陆个人真的是没有太搞明白。有时间要好好学习下postgresql。和正常自己了解的mysql等数据库比是完全的知识盲区了。有盲区，那就有时间学习一下了。加油......


