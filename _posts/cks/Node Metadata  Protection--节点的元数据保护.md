![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615602337342-4bff4851-aa6d-43dd-b3d7-8445c22ca29c.png#align=left&display=inline&height=291&margin=%5Bobject%20Object%5D&name=image.png&originHeight=582&originWidth=1129&size=451386&status=done&style=none&width=564.5)
## 1. 关于元数据
kubernets集群不管是运行与公有云还是私有云，都是有些元数据的资源的各种各样的标签。比如镜像id，网络设备id,硬盘的唯一id等。
## 2. 举一个例子
### 2.1 cloud platform node metadata   云平台节点元数据

1. 拿谷歌云和亚马逊云来说
1. 默认的情况下可以从虚拟机vm（云主机）访问元数据服务的api
1. 元数据中保护有vm节点（云主机）的各种凭据信息。如网络id,镜像id  vpcid.硬盘等待各种相关信息。具体详细度要看云商平台或者私有云架构
1. 可以包含诸如kubelet凭证之类的置备数据

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615602388327-1cb7dfaf-0e9e-47b5-a782-fec682a529f6.png#align=left&display=inline&height=290&margin=%5Bobject%20Object%5D&name=image.png&originHeight=579&originWidth=1123&size=203486&status=done&style=none&width=561.5)
### 2.2 access sensitive node metadta    访问敏感节点元数据的原则


2.2.1 最常说的权限控制原则

1.  Limitat  permissions for  instance credentials  权限最小化原则。
1. 确保cloud-instance-account仅具有必要的权限
1. 每个云提供商都应遵循一系列建议
1. 权限控制不在kubernetes中



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611198527011-965bfcc7-f496-4003-a563-241db654b944.png#align=left&display=inline&height=546&margin=%5Bobject%20Object%5D&name=image.png&originHeight=546&originWidth=985&size=127643&status=done&style=none&width=985)
## 3. restrict access using networkpolicies 使用网络策略限制访问
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615602533546-31982bdd-564f-441a-a432-7ff9bcd80cbf.png#align=left&display=inline&height=292&margin=%5Bobject%20Object%5D&name=image.png&originHeight=584&originWidth=1124&size=218635&status=done&style=none&width=562)


### 3.1 限制访问云商的元数据
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611198848485-76d6a1e3-be3e-493c-a4f9-6c4f9a3f9b33.png#align=left&display=inline&height=530&margin=%5Bobject%20Object%5D&name=image.png&originHeight=530&originWidth=1037&size=100777&status=done&style=none&width=1037)
由于没有谷歌云拿腾讯云意淫下了，可能理解的不是很对。往指教：
翻了下腾讯云的文档关于元数据也有文档：[https://cloud.tencent.com/document/product/213/4934?from=10680](https://cloud.tencent.com/document/product/213/4934?from=10680)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615604076329-124aab84-b2aa-45b8-b99d-3bc7c3c4a1c4.png#align=left&display=inline&height=86&margin=%5Bobject%20Object%5D&name=image.png&originHeight=172&originWidth=1435&size=30892&status=done&style=none&width=717.5)
就简单的证明一下，node节点和pod节点都可以访问云商的源数据。相对于谷歌云的文档，腾讯的还是略简单，想比着课程查询下硬盘，貌似还是没有这接口的。不过觉得下面这话说的很对，能访问实例就可以查看元数据。关于元数据的安全也很重要啊......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615604226553-c61c9b4b-5701-4b13-b6e5-abfad67ebe13.png#align=left&display=inline&height=102&margin=%5Bobject%20Object%5D&name=image.png&originHeight=203&originWidth=760&size=18526&status=done&style=none&width=380)
### 3.2通过networkpolicy 限制对元数据的访问
ping metadata.tencentyun.com得到medata的地址169.254.0.23，依然是命名空间级别的限制
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615604658950-f9b23606-0243-4a66-838c-c3a59e930ea7.png#align=left&display=inline&height=194&margin=%5Bobject%20Object%5D&name=image.png&originHeight=388&originWidth=701&size=21322&status=done&style=none&width=350.5)
```html
kubectl apply -f deny.yaml
kubectl -n metadata  exec nginx -it bash
curl http://metadata.tencentyun.com/latest/meta-data/instance/image-id
```
OK,如下图获取不了元数据中的镜像id了
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615604707387-06c049ce-39b4-4329-ae88-8d5adef69587.png#align=left&display=inline&height=75&margin=%5Bobject%20Object%5D&name=image.png&originHeight=149&originWidth=1267&size=22863&status=done&style=none&width=633.5)
然后我如何运行一组pod去访问元数据呢？
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615605242804-40fd4639-10de-40aa-977a-971b153d04fb.png#align=left&display=inline&height=218&margin=%5Bobject%20Object%5D&name=image.png&originHeight=436&originWidth=924&size=27125&status=done&style=none&width=462)
```html
kubectl apply -f allow.yaml
```
嗯 matchLabels我设置了一个不存在的，然后给pods加上labels
```html
kubectl get pods --show-labels -n metadata
kubectl label pod nginx role=metadata-accessor -n metadata
kubectl get pods --show-labels -n metadata
kubectl -n metadata  exec nginx -it bash
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615605679547-6dc6b7f8-b416-497d-89fa-0d4725dcfd45.png#align=left&display=inline&height=202&margin=%5Bobject%20Object%5D&name=image.png&originHeight=403&originWidth=1353&size=67371&status=done&style=none&width=676.5)
OK,可以返回元数据了。now现在去掉role=metadata-accessor 的标签
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615605877950-b7482970-14e7-4af9-bbbe-8c44265c37d7.png#align=left&display=inline&height=86&margin=%5Bobject%20Object%5D&name=image.png&originHeight=172&originWidth=1276&size=26426&status=done&style=none&width=638)
验证通过，其实我觉这节课主要的还是再强调networkpolicy。不仅仅是元数据的保护。networkpolicy是很重要的基石。




