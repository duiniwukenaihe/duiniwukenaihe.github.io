---
layout: post
title: 日志服务CLS对接Grafana
date: 2021-04-16 4:00:00
category: 腾讯云
tags: CLS kubernetes 日志服务 grafana
author: duiniwukenaihe
---
* content
{:toc}

# 背景：
[腾讯云CLB（负载均衡）与CLS（日志服务）集成](https://cloud.tencent.com/developer/article/1811695)。然后看日志服务CLS专栏有一篇
[CLS 对接 Grafana](https://cloud.tencent.com/developer/article/1785751)的博文。个人就也想尝试一下。当然了我的grafana是 [Prometheus-oprator](https://cloud.tencent.com/developer/article/1807805)方式搭建在kubernetes集群中的。详见：[https://cloud.tencent.com/developer/article/1807805](https://cloud.tencent.com/developer/article/1807805)。
下面开始记录下自己搭建的过程
# 一.  Grafana中的配置
参照[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)，但是饼形图的插件个人已经在[Prometheus-oprator](https://cloud.tencent.com/developer/article/1807805)的搭建过程中安装了。现在重要的就是按照cls的插件和更改grafana的配置
## 1. 关于Grafana cls插件的安装
插件安装的过程还是简单的
```
###查看grafana pods的名称
kubectl get pods -n monitoring
###进入grafana容器
kubectl exec -it grafana-57d4ff8cdc-8hwsq bash -n monitoring
###下载cls插件并解压
cd /var/lib/grafana/plugins/
wget https://github.com/TencentCloud/cls-grafana-datasource/releases/latest/download/cls-grafana-datasource.zip
unzip cls-grafana-datasource.zip
```


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618537833140-611d0851-fa89-4986-bb6e-84fdcfc8ecda.png#clientId=u4eef2b90-ea4a-4&from=paste&height=575&id=uad23986c&margin=%5Bobject%20Object%5D&originHeight=575&originWidth=1849&originalType=binary&size=120440&status=done&style=none&taskId=ud3379610-4f93-4d6a-ac89-468cfe7a417&width=1849)
至于版本的要求验证就忽略了....因为我的granfa image版本是7.4.3，是已经确认过的......。至于如何让grafana插件加载生效呢？我一遍都是用最笨的方法delete一下pod......
```
kubectl delete pods  grafana-57d4ff8cdc-8hwsq  -n monitoring
kubectl get pods -n monitoring
```
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618538462283-51a85958-950b-4a82-8888-0a2e5f76a458.png#clientId=u4eef2b90-ea4a-4&from=paste&height=637&id=u0f9c70ee&margin=%5Bobject%20Object%5D&originHeight=637&originWidth=1661&originalType=binary&size=95713&status=done&style=none&taskId=u85876567-7157-48fe-9dd6-55373cf5ad9&width=1661)
当然了这个时候如果进入grafana确认插件的安装成功与否一般的（grafana-cli plugins install通过官方源下载或者安装的）就可以了。但是安装cls这个插件是不可以的......为什么呢？强调一下腾讯云这个插件是一个**非官方认证的插件**。如果需要信任非官方的插件grafana是要开启配置参数的


## 2. 如何修改grafana的配置文件呢？
参照[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)。部署完了插件的安装，是要修改grafana.ini的配置文件的。仔细观察一下prometheus-operator中 grafana的配置文件是默认的，并没有其他方式进行挂载，那该怎么办呢？
参照：[https://blog.csdn.net/u010918487/article/details/110522133](https://blog.csdn.net/u010918487/article/details/110522133)
**将grafana的配置文件以configmap的方式进行挂载**
具体流程：
### 1.  将grafana容器中的grafana.ini文件复制到本地
就是复习一下kubectl cp命令了：
```
kubectl cp grafana-57d4ff8cdc-ms4z9:/etc/grafana/grafana.ini /root/grafana/grafana.ini  -n monitoring
```
然后本地修改grafana.ini配置文件：
嗯：   ;allow_loading_unsigned_plugins = 这一句配置修改为下面的这句
```
allow_loading_unsigned_plugins = tencent-cls-grafana-datasource
```
图中没有删除上面那句只是为了方便演示：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618538771607-12cfd696-f7fd-4e0b-bf48-51a44c184572.png#clientId=u4eef2b90-ea4a-4&from=paste&height=546&id=u324f6aac&margin=%5Bobject%20Object%5D&originHeight=546&originWidth=1037&originalType=binary&size=71447&status=done&style=none&taskId=u5807a560-96e8-47da-bafe-31b258d7877&width=1037)
这配置的作用就是开启tencent-cls-grafana-datasource这个非认证插件的加载。
### 2. 将修改后的grafana.ini以configmap的方式挂载到kubernetes集群
```
kubectl create cm grafana-config --from-file=`pwd`/grafana.ini -n monitoring
```
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539144911-fe95f91e-7c6d-4937-9370-7f8b9037ea11.png#clientId=u4eef2b90-ea4a-4&from=paste&height=486&id=u7f889c49&margin=%5Bobject%20Object%5D&originHeight=486&originWidth=1089&originalType=binary&size=45421&status=done&style=none&taskId=u988c6388-7be8-4054-99e2-3ac9508d9c0&width=1089)
### 3.  修改grafana-deployment.yaml挂载grafana-config
```
########volumeMounts部分新增以下内容：
      - mountPath: /etc/grafana
        name: grafana-config
        readOnly: true
########volumes部分新增以下内容：
    - configMap:
        name: grafana-config
      name: grafana-config
```
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539375287-c2978e09-caea-48e9-bd97-0d3a5e79adb3.png#clientId=u4eef2b90-ea4a-4&from=paste&height=430&id=ud875ea09&margin=%5Bobject%20Object%5D&originHeight=430&originWidth=922&originalType=binary&size=45323&status=done&style=none&taskId=u71292083-ea40-4b60-8ecb-85d8f693cca&width=922)
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539389360-5bccef56-1c71-4039-baf6-1ddae29b4f8c.png#clientId=u4eef2b90-ea4a-4&from=paste&height=146&id=u7cd11f71&margin=%5Bobject%20Object%5D&originHeight=146&originWidth=688&originalType=binary&size=10325&status=done&style=none&taskId=u7c4613da-ef28-453b-a5c9-648f309f787&width=688)
```
kubectl apply -f grafana-deployment.yaml -n monitoring
```
等待grafana pod 重新部署成功：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539473850-830889ce-56a4-460c-b49a-6d736f65ccb7.png#clientId=u4eef2b90-ea4a-4&from=paste&height=377&id=u8568eb8f&margin=%5Bobject%20Object%5D&originHeight=377&originWidth=1559&originalType=binary&size=60342&status=done&style=none&taskId=ud7d6269b-fedc-4f31-bc07-9db24635a8a&width=1559)
## 3. grafana dashboard中的配置
### 1. 在左侧导航栏中，单击【Creat Dashboards】。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539828892-cfe4ab52-0571-46bb-9305-e4eb53884ae2.png#clientId=u4eef2b90-ea4a-4&from=paste&height=734&id=ua6b0d625&margin=%5Bobject%20Object%5D&originHeight=734&originWidth=1472&originalType=binary&size=207780&status=done&style=none&taskId=u072de07e-2794-493f-9297-5452b264317&width=1472)
### 2. 在 Data Sources 页面，单击【Add data source】。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539842631-77611f02-154d-41aa-aff0-11daa8bf4409.png#clientId=u4eef2b90-ea4a-4&from=paste&height=856&id=uc9603f08&margin=%5Bobject%20Object%5D&originHeight=856&originWidth=1719&originalType=binary&size=98423&status=done&style=none&taskId=u296b8784-3aa8-42e2-acda-93d55bf7bcc&width=1719)
### 3. 选中【Tencent Cloud Log Service Datasource】，并按照如下说明配置数据源
 引用自：[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)
| **配置项** |  |
| --- | --- |
| Security Credentials | SecretId、SecretKey：API 请求密钥，用于身份鉴权。可前往 API 密钥管理 获取地址。 |
| Log Service Info | region：日志服务区域简称。例如，北京区域填写ap-beijing。完整的区域列表格式请参考 [地域列表。](https://cloud.tencent.com/document/product/614/18940)
Topic：日志主题ID 。 |

![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618540367090-8a1f319c-1271-4f9e-9c1c-b0bf45dd44ad.png#clientId=u4eef2b90-ea4a-4&from=paste&height=876&id=u51045433&margin=%5Bobject%20Object%5D&originHeight=876&originWidth=1577&originalType=binary&size=82720&status=done&style=none&taskId=ua76dd4d2-30e5-4398-8c5e-e4365c1fb6a&width=1577)
嗯登陆腾讯云后台cam控制台[https://console.cloud.tencent.com/cam](https://console.cloud.tencent.com/cam)。快速新建用户，新建一个名为cls的用户：登陆方式：编程访问，用户权限：[QcloudCLSReadOnlyAccess](https://console.cloud.tencent.com/cam/policy/detail/27395381&QcloudCLSReadOnlyAccess&2)，可接收消息类型全部就注释掉了。


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618539941565-654ee2f1-50fd-48ec-963c-1cbcb769429c.png#clientId=u4eef2b90-ea4a-4&from=paste&height=891&id=ub95f6b64&margin=%5Bobject%20Object%5D&originHeight=891&originWidth=1585&originalType=binary&size=130650&status=done&style=none&taskId=u647ec69f-aff1-4bde-b0a1-eb8d5987a9c&width=1585)




进行测试 sava&test。嗯  结果是显示操作未授权应该是下面的这个
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618540546556-72dd18cf-af68-47dc-b304-de3d88bec507.png#clientId=u4eef2b90-ea4a-4&from=paste&height=212&id=u7dac6e6e&margin=%5Bobject%20Object%5D&originHeight=212&originWidth=972&originalType=binary&size=72755&status=done&style=none&taskId=ua96b502c-fd16-48ae-ba78-df2ea150cc6&width=972)
果断进入腾讯云后台加上了[QcloudCLSFullAccess](https://console.cloud.tencent.com/cam/policy/detail/534803&QcloudCLSFullAccess&2)的权限


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618540556330-39c6ba28-1e02-4a13-98cd-59b092ea5988.png#clientId=u4eef2b90-ea4a-4&from=paste&height=366&id=u546a5025&margin=%5Bobject%20Object%5D&originHeight=366&originWidth=1423&originalType=binary&size=36116&status=done&style=none&taskId=u6a63f636-ea0b-453f-9697-bce7b39637d&width=1423)
再运行sava&test，ok成功。但是个人对权限比较敏感，这样的fullaccess的 都比较怕...腾讯云官方开源的好多组件对权限的声明都不是那么的多。 也希望能明确一下权限的边界。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618540627163-1a94b37d-c776-43f7-a66f-0bbeb8061171.png#clientId=u4eef2b90-ea4a-4&from=paste&height=553&id=u14fee950&margin=%5Bobject%20Object%5D&originHeight=553&originWidth=1089&originalType=binary&size=88023&status=done&style=none&taskId=ue004abc3-2a5f-4e7f-915d-2a080180f87&width=1089)
### 4. 配置 dashboard

1. 在左侧导航栏中，单击【**Creat Dashboards**】。
1. 在 Dashboard 页面，单击【**Add new panel**】。
1. 将数据源选择为您新建的日志数据源。如下图所示：

![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618542858288-0f8d250a-592b-40f2-8ada-1cc0bbadd8ae.png#clientId=u502b8fb1-00b2-4&from=paste&height=852&id=u58e68a0a&margin=%5Bobject%20Object%5D&originHeight=852&originWidth=1191&originalType=binary&size=61175&status=done&style=none&taskId=u9b7265e6-a20f-40e9-945c-d5325c020e1&width=1191)
一下图片引用自[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543108928-ca8bedee-76d8-47ed-9697-5533aa366348.png#clientId=u502b8fb1-00b2-4&from=paste&height=370&id=u2e7d55b9&margin=%5Bobject%20Object%5D&originHeight=397&originWidth=747&originalType=binary&size=38997&status=done&style=none&taskId=u0d67fbd7-9c87-406d-9861-ec2cac4d89c&width=697)
## 4. 展示图表
 本来图表都该在上面搞完的。但是个人觉得还是单独列出来吧
### 1. 时间折线图
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543418635-236df0ed-403e-41a1-ad6d-11347bde6104.png#clientId=u502b8fb1-00b2-4&from=paste&height=869&id=u2878ebe3&margin=%5Bobject%20Object%5D&originHeight=869&originWidth=1415&originalType=binary&size=135199&status=done&style=none&taskId=u0be96ba5-c423-4fe0-9191-fb00893fb39&width=1415)
输入的 Query 语句如下所示：
```
* | select histogram( cast(__TIMESTAMP__ as timestamp),interval 1 minute) as time, count(*) as pv,count( distinct remote_addr) as uv group by time order by time limit 1000
```

- Format：选择 **Graph,Pie,Gauge panel**。
- Metrics：**pv，uv**。
- Bucket：无聚合列，**不填写**。
- Time : **time**。

![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543583032-85e54d36-3329-4749-8fb7-38eb60d2a3d1.png#clientId=u502b8fb1-00b2-4&from=paste&height=862&id=ue4718909&margin=%5Bobject%20Object%5D&originHeight=862&originWidth=1498&originalType=binary&size=163030&status=done&style=none&taskId=u7d5f524c-1f63-4ea7-9251-dfff9daec58&width=1498)
记得在Visualization中选择折线图。 Graph 和Time Series两个出来的是一样的...也没有搞清楚区别.
### 2. 饼形图
搞不出来一般就是没有安装饼形图插件吧。一定记得前提安装了饼形图插件

- 输入的 Query 语句如下所示：
```
* | select count(*) as count, status group by status
```

- Format：选择 **Graph,Pie,Gauge panel**。
- Metrics：**count**。
- Bucket：**status**。
- Time：不是连续时间数据，**不填写**。

![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543828487-350a4cb8-64d1-4e8c-a775-5b4e33c12542.png#clientId=u502b8fb1-00b2-4&from=paste&height=882&id=u7f320ba4&margin=%5Bobject%20Object%5D&originHeight=882&originWidth=1465&originalType=binary&size=159764&status=done&style=none&taskId=ua7038481-4e29-4dfb-8b60-d951011660e&width=1465)
不管那种图表都的在Visualization选择要展现的形式啊...
### 3. 柱状图，压力图
柱状图，压力图（bar gauge）统计访问延时前10的页面
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543918658-66859c69-69c1-4805-8312-ffa8ca21a1b9.png#clientId=u502b8fb1-00b2-4&from=paste&height=810&id=uabd702c2&margin=%5Bobject%20Object%5D&originHeight=810&originWidth=1465&originalType=binary&size=139750&status=done&style=none&taskId=u7516b39d-1756-4039-bf4f-55c11b6483b&width=1465)

- 输入的 Query 语句如下所示：
```
* | select http_referer,avg(request_time) as lagency group by http_referer order by lagency desc limit 10
```

- Format：选择** Graph,Pie,Gauge panel**。
- Metrics：lagency。
- Bucket：http_referer。
- Time：不是连续时间数据，**不填写**

为什么我做出来跟[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)中的不一样呢？
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618543981429-68d59010-1e63-42e3-9eed-4090ebcc5e8b.png#clientId=u502b8fb1-00b2-4&from=paste&height=370&id=u23fd1e04&margin=%5Bobject%20Object%5D&originHeight=370&originWidth=732&originalType=binary&size=94016&status=done&style=none&taskId=u72ff6608-68e8-44c9-8777-eae6aa9cb1b&width=732)
参照[http://codetd.com/article/11171054](http://codetd.com/article/11171054)
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618544047141-cdff7824-a4f2-4d73-8a65-9023c235d30a.png#clientId=u502b8fb1-00b2-4&from=paste&height=885&id=u12ab3b30&margin=%5Bobject%20Object%5D&originHeight=885&originWidth=1258&originalType=binary&size=202886&status=done&style=none&taskId=u8d7d0ee6-57eb-410c-bee6-8ed5c4835b6&width=1258)
可能cls官方这文章木有考虑我太小白....，按照大佬的文章修改下OK了
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618546067976-7b5ca78f-e1c9-48bf-8baf-d2886a4865a6.png#clientId=ud7da4c3b-34fe-4&from=paste&height=844&id=uc2065c3d&margin=%5Bobject%20Object%5D&originHeight=844&originWidth=1446&originalType=binary&size=177835&status=done&style=none&taskId=uc8ffa5c6-85c0-4b09-9f45-52a49f69295&width=1446)
但是 颜色是不是也搞的绚丽点？**Color scheme Thresholds 两个的配置可以满足你的需要....哈哈哈**


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618545951564-600a4b56-2574-4279-a0c0-3466d1679a48.png#clientId=ud7da4c3b-34fe-4&from=paste&height=733&id=u116e5ecb&margin=%5Bobject%20Object%5D&originHeight=733&originWidth=1326&originalType=binary&size=153926&status=done&style=none&taskId=ue894d4f3-644d-4c04-8568-c8105f7d6f1&width=1326)
### 4. 表格Table
表格的应该就算是简单的了
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618544698729-2153b639-8888-443c-b814-2fbaa026d8d0.png#clientId=u502b8fb1-00b2-4&from=paste&height=848&id=uaab884e5&margin=%5Bobject%20Object%5D&originHeight=848&originWidth=1384&originalType=binary&size=121900&status=done&style=none&taskId=uaaa6ac7e-451f-4621-9062-2789ad976e9&width=1384)

- 输入的 Query 语句如下所示：
```
* | select remote_addr,count(*) as count group by remote_addr order by count desc limit 10
```

- Format：Table
## 5. 最终效果
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618545902421-68e93cef-f787-4093-a2b7-31d95fca1383.png#clientId=ud7da4c3b-34fe-4&from=paste&height=704&id=u61a528ae&margin=%5Bobject%20Object%5D&originHeight=704&originWidth=1387&originalType=binary&size=164213&status=done&style=none&taskId=u6ea479ff-0751-48ce-bb94-cf6dd98d20b&width=1387)
# 鸣谢：
### [日志服务CLS小助手](https://cloud.tencent.com/developer/user/1482442)：[https://cloud.tencent.com/developer/article/1785751](https://cloud.tencent.com/developer/article/1785751)
### 代码天地： [http://codetd.com/article/11171054](http://codetd.com/article/11171054)
### [katy的小乖](https://blog.csdn.net/u010918487):[  https://blog.csdn.net/u010918487/article/details/110522133](https://blog.csdn.net/u010918487/article/details/110522133)
