---
layout: post
title: 2021-06-22-Kubernetes traefik tls证书到期替换
date: 2021-06-22 2:00:00
category: kubernetes
tags: traefik tls
author: duiniwukenaihe
---
* content
{:toc}
# 背景：

ssl证书都是一年签发的。到了六月份了一年一度的证书替换的日子到了.......。过去的方法一直都是先delete secret，然后继续创建一个新的。这次就突发奇想的，还有其他方法吗......百度了一下还真的搜索到了：

[https://blog.csdn.net/cyxinda/article/details/107854881](https://blog.csdn.net/cyxinda/article/details/107854881)

我的证书是在腾讯云上面购买的 TrustAsia 域名型(DV)通配符(1 年)的 ssl证书。

![image.png](/assets/images/2021/06-22/fgxd81atai.png)

考虑到成本毕竟申请的dv的泛域名证书。关于dv ov证书的区别：

![image.png](/assets/images/2021/06-22/7oea53q8uj.png)

```
                                             此图片来源于网络
```

# 关于secret tls的创建方式

关于secret可以参照[https://kubernetes.io/zh/docs/concepts/configuration/secret/](https://kubernetes.io/zh/docs/concepts/configuration/secret/)。kubernetes官方文档。这里就直接创建一下tls secret 了

在腾讯云平台ssl管理页面[https://console.cloud.tencent.com/ssl](https://console.cloud.tencent.com/ssl)

![image.png](/assets/images/2021/06-22/vjtkctrqtf.png)

找到相关证书并点击下载将证书下载到本地。

将证书上传到服务器，并解压zip文件。解压后的文件列表如下

![image.png](/assets/images/2021/06-22/7d7lrkt9ft.png)

我是直接进入Nginx文件夹，文件列表如下：

![image.png](/assets/images/2021/06-22/b8ujurpt4q.png)

当然了 其他平台申请的证书可能不是这样的（记得阿里云的不是这样的来，可以openssl 转一下吧？）

```
 kubectl create secret tls all-xxx-com --key=2_xxx.com.key --cert=1_xxx.com_bundle.crt -n master
```

traefik中应用使用证书可以参照：[traefik2 安装实现 http https](https://duiniwukenaihe.github.io/2019/10/17/k8s-traefik2/),[2019-12-27-traefik](https://duiniwukenaihe.github.io/2019/12/27/traefik/).下面进入正题切换修改到期证书...

注：以下参考了[https://blog.csdn.net/cyxinda/article/details/107854881](https://blog.csdn.net/cyxinda/article/details/107854881)

# 几种更换kubernetes中tls证书的方式：

## 1.删除再重建

这应该是最常见的...反正我过去是经常用

```
kubectl delete secret all-xxx.com -n master
kubectl create secret tls all-xxx-com --key=2_xxx.com.key --cert=1_xxx.com_bundle.crt -n master
```

## 2. 通过--dry-run参数预览，然后apply

```
kubectl create secret tls all-xxx-com --key=2_xxx.com.key --cert=1_xxx.com_bundle.crt -n master --dry-run -o yaml |kubectl apply -f -
```

算是复习了一下--dry-run的命令。也比较优雅。通过apply更新 secret。

## 3. 我可不可以edit 修改一下 secret中秘钥的内容

```
kubectl edit secret all-xxxcom -n master
```

![image.png](/assets/images/2021/06-22/fw0triyuoj.png)

```
base64 1_xxx.com_bundle.crt得到的内容将secret tls.crt中内容替换
base64 2_xxx.com.key得到的内容将secret中的tls.key的内容替换
```

## 4. 更优雅的方式：

![image.png](/assets/images/2021/06-22/h6ywnpfkmx.png)

个人来说倾向于第二种方式还是......

# 总结一下：

1. 删除重建是最笨的方式，当然了不会其他方式也可以用这种方式。在删除新建secret的空窗期，是存在风险的
2. kubernetes api可以通过json或者yaml文件对元数据进行更新或者交互
3. --dry-run 的参数还是很有用的
4. secret还是基于namespace。同一个secret在好多命名空间都有同样的存在，有没有较为方便的管理方式呢？