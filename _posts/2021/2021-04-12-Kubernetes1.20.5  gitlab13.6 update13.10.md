---
layout: post
title: Kubernetes1.20.5  gitlab13.6 update13.10
date: 2021-04-11 7:00:00
category: kubernetes1.20
tags: kubernetes gitlab
author: duiniwukenaihe
---
* content
{:toc}

安装的时候gitlab版本用的13.6版本....貌似最近有bug了 看好多人都建议升级到13.10版本后
流氓了一下 只修改了giltlab.yaml中image的tag  然后
```
kuberctl apply -f gitlab.yaml
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618216978918-02dff250-7360-4b93-97bd-7026a37830f3.png#align=left&display=inline&height=199&margin=%5Bobject%20Object%5D&name=image.png&originHeight=397&originWidth=836&size=49954&status=done&style=none&width=418)
然后
```
kubectl describe pods gitlab-b9d95f784-dkgbj -n kube-ops
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618217282017-e7bc494c-07a6-4aa4-a852-2b69aa3e8e30.png#align=left&display=inline&height=262&margin=%5Bobject%20Object%5D&name=image.png&originHeight=524&originWidth=1708&size=71077&status=done&style=none&width=854)
嗯  pvc挂不上啊 这样...最笨的方法吧。删除重建一下
```
kubectl delete -f gitlab.yaml -n kube-ops
kubectl apply -f gitlab.yaml -n kube-ops
kubectl describe pods  gitlab-b9d95f784-7h8dt -n kube-ops
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618217425658-426053cb-1ff3-480a-b942-dad7594a059d.png#align=left&display=inline&height=288&margin=%5Bobject%20Object%5D&name=image.png&originHeight=575&originWidth=1401&size=88676&status=done&style=none&width=700.5)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618217520144-8e4517d3-c579-4f6e-83ca-d2432c93f39f.png#align=left&display=inline&height=161&margin=%5Bobject%20Object%5D&name=image.png&originHeight=322&originWidth=1107&size=49981&status=done&style=none&width=553.5)
登陆web验证一下....
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618217597846-76819481-b586-42cb-a7d8-051f58e8ee8f.png#align=left&display=inline&height=324&margin=%5Bobject%20Object%5D&name=image.png&originHeight=647&originWidth=1839&size=95012&status=done&style=none&width=919.5)
注：正常的版本升级还是参照这里的文档 [https://github.com/sameersbn/docker-gitlab](https://github.com/sameersbn/docker-gitlab)。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618217884465-e4fad0c9-726e-460b-9351-177997d12f51.png#align=left&display=inline&height=402&margin=%5Bobject%20Object%5D&name=image.png&originHeight=805&originWidth=1605&size=111990&status=done&style=none&width=802.5)
我的这个gitlab首先基本是空的，而且也是小版本升级。就忽略了。尤其是牵扯到数据的，别想着走什么捷径......安全第一哈哈哈哈。也警戒以下自己。莫取巧。
