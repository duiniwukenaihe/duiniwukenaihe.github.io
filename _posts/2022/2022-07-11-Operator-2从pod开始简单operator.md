---
layout: post
title: 2022-07-11-Operator-2从pod开始简单operator
date: 2022-07-11 2:00:00
category: kubernetes
tags: kubernetes operator
author: duiniwukenaihe
---
* content
{:toc}

# 背景：
前置内容：[Operator-1初识Operator](https://www.yuque.com/duiniwukenaihe/hg6ymd/nf8ghd#i5zPG)，从pod开始简单创建operator......
# 创建Pod
## RedisSpec 增加Image字段
恩 强调一下 我故意在api/v1/redis_type.go中加一个字段。接下来该怎么操作呢？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656649196625-e944870a-83b0-42cd-9168-6bda6bac44af.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=637&id=u3447f072&margin=%5Bobject%20Object%5D&name=image.png&originHeight=573&originWidth=1600&originalType=binary&ratio=1&rotation=0&showTitle=false&size=154513&status=done&style=none&taskId=u560b78a3-21b4-4ea9-9a31-650605a4a3c&title=&width=1777.777824872807)

## 先发布了一下crd
```
[zhangpeng@zhangpeng kube-oprator1]$ ./bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
[zhangpeng@zhangpeng kube-oprator1]$ kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/redis.myapp1.zhangpeng.com configured
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656649364607-2d09f950-ee6f-4bf0-94cf-da6aece1b5b5.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1028&id=u96626f2e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=925&originWidth=1725&originalType=binary&ratio=1&rotation=0&showTitle=false&size=246512&status=done&style=none&taskId=u5ed69358-bed0-4e4d-aa0f-990c5ec6649&title=&width=1916.666717440995)
describe crd 配置里面已经有了Image字段！
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl describe crd redis.myapp1.zhangpeng.com 
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656649436885-5215972a-b348-4304-a0cb-0436991898f9.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=910&id=u62383478&margin=%5Bobject%20Object%5D&name=image.png&originHeight=819&originWidth=1578&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103121&status=done&style=none&taskId=u740afb59-880b-4d00-9e60-d387636c800&title=&width=1753.333379780806)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656649526029-ec1c669f-5613-4421-a191-ae3cbb56d193.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1043&id=ud394dded&margin=%5Bobject%20Object%5D&name=image.png&originHeight=939&originWidth=1467&originalType=binary&ratio=1&rotation=0&showTitle=false&size=168874&status=done&style=none&taskId=ueca0b009-a413-4df5-b0df-39607e7806b&title=&width=1630.0000431802548)
## 镜像要不要发布一下？
发布镜像的时候**SetupWebhookWithManager**打开了....不打开貌似发布了会报错
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656662419248-884d3972-d128-4317-80fa-03e0848f1254.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=677&id=u61f24eae&margin=%5Bobject%20Object%5D&name=image.png&originHeight=609&originWidth=1598&originalType=binary&ratio=1&rotation=0&showTitle=false&size=161414&status=done&style=none&taskId=u7d93da73-d551-4b61-a92b-37cde2e6e2c&title=&width=1775.555602591716)
**打包镜像image并发布**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656898939921-8892db28-be7b-4114-a4ef-d151e46bb2db.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=463&id=u2bc98baf&margin=%5Bobject%20Object%5D&name=image.png&originHeight=417&originWidth=1202&originalType=binary&ratio=1&rotation=0&showTitle=false&size=61559&status=done&style=none&taskId=u7b211411-71cd-4deb-9deb-68cf5e96b2c&title=&width=1335.5555909356963)
```
[zhangpeng@zhangpeng kube-oprator1]$ cd config/manager &&  kustomize edit set image controller=ccr.ccs.tencentyun.com/layatools/zpredis:v2
[zhangpeng@zhangpeng manager]$ cd ../../
[zhangpeng@zhangpeng kube-oprator1]$ kustomize build config/default | kubectl apply -f -

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656658353438-578c9066-0099-4624-9a6f-23cfab64f193.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=291&id=u99770ba0&margin=%5Bobject%20Object%5D&name=image.png&originHeight=262&originWidth=1374&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66292&status=done&style=none&taskId=u60e536cd-2365-4a3f-9982-1abec0a11af&title=&width=1526.666707109523)
注：当然了能使用原生的命令更好， 我这系统还是有问题的
## 继续强调一下SetupWebhookWithManager
发布完成继续关了main.go中 **SetupWebhookWithManage**r，否则没法**make run**本地调试哦。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656662601632-62f3d364-61f7-4b51-ad22-53ab4b6e2064.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=631&id=uad08117c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=568&originWidth=1608&originalType=binary&ratio=1&rotation=0&showTitle=false&size=162947&status=done&style=none&taskId=u851f9452-77a5-4eef-8709-19adeca7664&title=&width=1786.666713997171)
## 创建CreateRedis函数
helper/help_redis.go
```
package helper

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	v1 "kube-oprator1/api/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
)


func CreateRedis(client client.Client, redisConfig *v1.Redis) error {

	newpod := &corev1.Pod{}
	newpod.Name = podName
	newpod.Namespace = redisConfig.Namespace
	newpod.Spec.Containers = []corev1.Container{
		{
			Name:           redisConfig.Name,
			Image:           redisConfig.Spec.Image,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Ports: []corev1.ContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	return client.Create(context.Background(), newpod)

}
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656662784265-fda91dda-0a6d-4ad0-ba47-59557c043c0d.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=898&id=u812e4bd7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=808&originWidth=1516&originalType=binary&ratio=1&rotation=0&showTitle=false&size=171561&status=done&style=none&taskId=ubc1b3467-cf1b-4b0f-a821-4e4ffb12f9d&title=&width=1684.4444890669847)
## Reconcile调用CreateRedis函数
**controlers/redis_controller.go**
```
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)
	redis := &myapp1v1.Redis{}
	if err := r.Get(ctx, req.NamespacedName, redis); err != nil {
		fmt.Println(err)
	} else {
		fmt.Println("object", redis)
		err := helper.CreateRedis(r.Client, redis)
		return ctrl.Result{}, err

	}

	return ctrl.Result{}, nil
}
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656925503920-e23b430d-3918-4fa5-a7d2-1d106471320a.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=736&id=u8f8756e9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=662&originWidth=1659&originalType=binary&ratio=1&rotation=0&showTitle=false&size=191040&status=done&style=none&taskId=uacdc6c2c-da21-4077-b99e-cc2ae3dd80f&title=&width=1843.3333821649917)
## make run test
注：其实也可以不make run了......都发布到集群中了， 可以到集群中查看operator日志了！
test/redis.go
```
apiVersion: myapp1.zhangpeng.com/v1
kind: Redis
metadata:
  name: zhangpeng1
spec:
  port: 6379
  image: redis:latest
```
注意：特意搞了两个又创建了一个name为zhangpeng2的应用！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656663859019-5febd445-12f2-4a07-8982-3d98a63f0cb9.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=861&id=ud36ab4c4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=775&originWidth=1649&originalType=binary&ratio=1&rotation=0&showTitle=false&size=179426&status=done&style=none&taskId=u4297fc6a-2d04-45ae-ac46-f5c0811795a&title=&width=1832.2222707595367)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656663843817-f77c606f-647f-4c86-98cd-6f2f2fba7067.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=808&id=u760271b6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=727&originWidth=1659&originalType=binary&ratio=1&rotation=0&showTitle=false&size=301292&status=done&style=none&taskId=ud87a13ea-0a26-4f55-9ca3-d01a7646a2a&title=&width=1843.3333821649917)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656663893112-d4d073ff-62f8-447c-abd4-34acddbcc539.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=922&id=u5d63a456&margin=%5Bobject%20Object%5D&name=image.png&originHeight=830&originWidth=1794&originalType=binary&ratio=1&rotation=0&showTitle=false&size=428481&status=done&style=none&taskId=u45ef45a0-13ad-44f6-8213-bcc93f49829&title=&width=1993.3333861386348)
```
[zhangpeng@zhangpeng ~]$ kubectl get Redis
[zhangpeng@zhangpeng ~]$ kubectl get pods
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656663925048-a4713616-df62-4e3a-89a2-56549c537054.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=211&id=u64a7ff20&margin=%5Bobject%20Object%5D&name=image.png&originHeight=190&originWidth=1336&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37243&status=done&style=none&taskId=ua107e574-c82a-4ed2-8cfa-1ca3e9f6950&title=&width=1484.4444837687938)
创建成功！接着而来的是pod删除的问题：
```
[zhangpeng@zhangpeng ~]$ kubectl delete Redis zhangpeng2
[zhangpeng@zhangpeng ~]$ kubectl get pod
```
恩删除了zhangpeng2 Redis，but pod资源还存在，该怎么处理，让pod资源一起删除呢？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656664227461-7fd00074-e6d9-408f-b188-9bdc730dd4af.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=169&id=u6a5fff1f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=152&originWidth=1170&originalType=binary&ratio=1&rotation=0&showTitle=false&size=51172&status=done&style=none&taskId=u9695eec5-e7eb-43b2-8a43-fe06f3c84e0&title=&width=1300.0000344382402)
# POD资源删除
参照：[https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/)
Finalizers
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656665664159-6160c951-90bd-44fc-b963-92504b646ebd.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=726&id=u2945aaed&margin=%5Bobject%20Object%5D&name=image.png&originHeight=653&originWidth=1632&originalType=binary&ratio=1&rotation=0&showTitle=false&size=233581&status=done&style=none&taskId=ua7832051-f196-4244-a097-89554f7cd72&title=&width=1813.3333813702632)
## 清理资源
清理一下资源删除webhook Redis 等相关资源
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get ValidatingWebhookConfiguration
NAME                                             WEBHOOKS   AGE
cert-manager-webhook                             1          3d23h
kube-oprator1-validating-webhook-configuration   1          3d23h
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete ValidatingWebhookConfiguration
error: resource(s) were provided, but no name was specified
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete ValidatingWebhookConfiguration kube-oprator1-validating-webhook-configuration 
validatingwebhookconfiguration.admissionregistration.k8s.io "kube-oprator1-validating-webhook-configuration" deleted
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get MutatingWebhookConfiguration
NAME                                           WEBHOOKS   AGE
cert-manager-webhook                           1          3d23h
kube-oprator1-mutating-webhook-configuration   1          3d23h
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete MutatingWebhookConfiguration kube-oprator1-mutating-webhook-configuration 
mutatingwebhookconfiguration.admissionregistration.k8s.io "kube-oprator1-mutating-webhook-configuration" deleted
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis
NAME         AGE
zhangpeng3   5m
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete Redis zhangpeng3
redis.myapp1.zhangpeng.com "zhangpeng3" deleted
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete pods zhangpeng3
Error from server (NotFound): pods "zhangpeng3" not found
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods
No resources found in default namespace.

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656936868776-265f41b9-d307-4d08-b005-4b763e456f12.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=528&id=ue9c7025e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=475&originWidth=1570&originalType=binary&ratio=1&rotation=0&showTitle=false&size=128357&status=done&style=none&taskId=ub0db7085-7079-473c-8842-cb9cf378425&title=&width=1744.4444906564418)
这样作吧，还是为了本地调试方便，否则一些本地的修改，增加字段就要重新打包，发布，太麻烦了。老老实实make run 本地调试模式，待调试完成后再发布应用到集群，这才是正常的流程！
需求: 删除Redis 对应pod 可以删除，pod可以多副本。zhangpeng-0 zhangpeng-1  zhangpeng-3的顺序。更新策略，缩容策略。恩这是我想到的！当然了pod如果使用deployment 或者statefulset会更简单一些。这里还是拿pod演示了！
## RedisSpec增加副本数量字段
修改api/v1/redis_types.go中RedisSpec struct,增加如下配置：
```
	//+kubebuilder:validation:Minimum:=1
	//+kubebuilder:validation:Maximum:=5
	Num int `json:"num,omitempty"`
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656988840532-b1f6e17b-305b-4bc0-8d61-911e64bb9f91.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=480&id=ub21b2109&margin=%5Bobject%20Object%5D&name=image.png&originHeight=432&originWidth=1100&originalType=binary&ratio=1&rotation=0&showTitle=false&size=106802&status=done&style=none&taskId=u6de07c42-ee81-4c5c-8701-152bc32491e&title=&width=1222.2222546000548)
num字段为pod副本数量字段，依着port配置把副本数量做了一个1-5的限制！
继续发布一下crd：
```
[zhangpeng@zhangpeng kube-oprator1]$ ./bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
[zhangpeng@zhangpeng kube-oprator1]$ kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/redis.myapp1.zhangpeng.com configured
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656991861831-13aba9fd-aaa4-4a59-a49d-8ff586b049e8.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=964&id=u2cdb217b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=868&originWidth=1683&originalType=binary&ratio=1&rotation=0&showTitle=false&size=231131&status=done&style=none&taskId=u4beedc1c-1dcd-4f93-a66c-0aa36122194&title=&width=1870.0000495380839)
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl describe crd redis.myapp1.zhangpeng.com
```
确认crd中产生对应字段：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1656991991275-3499490d-a609-4d20-8871-56c0017e3130.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=824&id=u4d2bfda8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=742&originWidth=1233&originalType=binary&ratio=1&rotation=0&showTitle=false&size=92463&status=done&style=none&taskId=u244e1d9c-0770-444b-a305-f74611c672d&title=&width=1370.0000362926069)
## help_redis.go
修改helper/help_redis.go如下：

```
package helper

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	v1 "kube-oprator1/api/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
)
//拼装Pod名称列表
func GetRedisPodNames(redisConfig *v1.Redis) []string {
	podNames := make([]string, redisConfig.Spec.Num)
	for i := 0; i < redisConfig.Spec.Num; i++ {
		podNames[i] = fmt.Sprintf("%s-%d", redisConfig.Name, i)
	}
	fmt.Println("podnames:", podNames)
	return podNames
}
//判断对应名称的pod在finalizer中是否存在
func IsExist(podName string, redis *v1.Redis) bool {
	for _, pod := range redis.Finalizers {
		if podName == pod {
			return true
		}
	}
	return false

}

//创建pod
func CreateRedis(client client.Client, redisConfig *v1.Redis, podName string) (string, error) {
	if IsExist(podName, redisConfig) {
		return "", nil
	}

	newpod := &corev1.Pod{}
	newpod.Name = podName
	newpod.Namespace = redisConfig.Namespace
	newpod.Spec.Containers = []corev1.Container{
		{
			Name:            podName,
			Image:           redisConfig.Spec.Image,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Ports: []corev1.ContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	return podName, client.Create(context.Background(), newpod)

}

```
## 修改controllers/redis_controller.go：
```
import (
	"context"
	"fmt"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/tools/record"
	myapp1v1 "kube-oprator1/api/v1"
	"kube-oprator1/helper"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
)
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	redis := &myapp1v1.Redis{}
	if err := r.Client.Get(ctx, req.NamespacedName, redis); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}
  //取得需要创建的POD副本的名称
	podNames := helper.GetRedisPodNames(redis)
	fmt.Println("podNames,", podNames)
	updateFlag := false
  //删除POD，删除Kind:Redis过程中，会自动加上DeletionTimestamp字段，据此判断是否删除了自定义资源
	if !redis.DeletionTimestamp.IsZero() {
		fmt.Println(ctrl.Result{}, r.clearRedis(ctx, redis))
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}
  //创建POD
	for _, podName := range podNames {
		finalizerPodName, err := helper.CreateRedis(r.Client, redis, podName)
		if err != nil {
			fmt.Println("create pod failue,", err)
			return ctrl.Result{}, err
		}
		if finalizerPodName == "" {
			continue
		}
    //若该pod已经不在finalizers中，则添加
		redis.Finalizers = append(redis.Finalizers, finalizerPodName)
		updateFlag = true
	}
  //更新Kind Redis状态
	if updateFlag {
		err := r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}
//删除POD逻辑
func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myapp1v1.Redis) error {
	//从finalizers中取出podName，然后执行删除
	for _, finalizer := range redis.Finalizers {
		//删除pod
		err := r.Client.Delete(ctx, &v1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      finalizer,
				Namespace: redis.Namespace,
			},
		})
		if err != nil {
			return err
		}
	}
  //清空finializers，只要它有值，就无法删除Kind
	redis.Finalizers = []string{}
	return r.Client.Update(ctx, redis)
}

```
## make run & test
控制台运行make run&&控制台2运行apply命令：

```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl apply -f test/redis.yaml 
redis.myapp1.zhangpeng.com/zhangpeng3 configured
```
test/redis.yaml
```
apiVersion: myapp1.zhangpeng.com/v1
kind: Redis
metadata:
  name: zhangpeng3
spec:
  port: 6379
  num: 2
  image: redis:latest
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657001379999-057aae85-a4a0-44ff-8b34-e740e51e04aa.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=972&id=uef815280&margin=%5Bobject%20Object%5D&name=image.png&originHeight=875&originWidth=1434&originalType=binary&ratio=1&rotation=0&showTitle=false&size=167655&status=done&style=none&taskId=uc957edf3-caf8-44a2-9f9a-21c978a9166&title=&width=1593.3333755422532)
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods
[zhangpeng@zhangpeng kube-oprator1]$ kubectl apply -f test/redis.yaml 
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods

[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis/zhangpeng3 -o yaml

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657002119782-a38369d2-86cb-44d1-b5a9-527fa7a698c9.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=998&id=u819343f5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=898&originWidth=1591&originalType=binary&ratio=1&rotation=0&showTitle=false&size=174844&status=done&style=none&taskId=u623f392b-b728-48e4-a6f4-b0317a9bf1c&title=&width=1767.7778246078974)
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete test/redis.yaml 
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657002144970-3db32853-8b39-411d-8f34-45e61471c7f4.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=679&id=uab37a8b8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=611&originWidth=1338&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103702&status=done&style=none&taskId=ud322edf1-eb0e-40e8-9d72-83e11a4fc19&title=&width=1486.6667060498849)
## 紧接着问题又来了：
**创建副本为2：**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657002398485-505e7915-bbf6-4f31-a762-18f162365866.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=937&id=u3898f46d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=843&originWidth=1326&originalType=binary&ratio=1&rotation=0&showTitle=false&size=164335&status=done&style=none&taskId=u28ca5ff5-00e9-414d-b568-bb5a44ed1ca&title=&width=1473.3333723633389)
**修改副本数为3：**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657002430587-71ee5f21-3820-4d4f-a9ed-a9b1c20c2b3b.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1008&id=u01c36648&margin=%5Bobject%20Object%5D&name=image.png&originHeight=907&originWidth=1394&originalType=binary&ratio=1&rotation=0&showTitle=false&size=204101&status=done&style=none&taskId=u2ef0d178-90e0-4d7e-bba1-361dcaa45af&title=&width=1548.8889299204332)
紧接着缩容副本为2,会出现怎么操作呢？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657002481380-20d24a0f-2415-4591-90a3-ce1618721e48.png#clientId=ufffb9314-8eb8-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=967&id=u29465bd9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=870&originWidth=1390&originalType=binary&ratio=1&rotation=0&showTitle=false&size=185229&status=done&style=none&taskId=ubfe645d1-ac51-470f-ae35-920315d6876&title=&width=1544.444485358251)
没有自动去调度，现在需要一个自动缩容的方法......
# Pod资源缩容
pod的标签定义的是zhangpeng-0 zhangpeng-1 zhangpeng-2的类型，扩容是 0  1  2  3 4的递增模式，缩容也从4 3 2 1 0依次去删除 pod
注意：默认情况下前面一步的Redis资源已经删除(kubectl delete -f test/redis.yaml)
## Reconcile修改
controllers/redis_controller.go 中Reconcile修改为如下：
```
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	redis := &myapp1v1.Redis{}
	if err := r.Client.Get(ctx, req.NamespacedName, redis); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		fmt.Println("object", redis)
	}
	if !redis.DeletionTimestamp.IsZero() || len(redis.Finalizers) > redis.Spec.Num {
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}
	if !redis.DeletionTimestamp.IsZero() || len(redis.Finalizers) > redis.Spec.Num {
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}
	podNames := helper.GetRedisPodNames(redis)
	fmt.Println("podNames,", podNames)
	updateFlag := false
	for _, podName := range podNames {
		finalizerPodName, err := helper.CreateRedis(r.Client, redis, podName)
		if err != nil {
			fmt.Println("create pod failue,", err)
			return ctrl.Result{}, err
		}
		if finalizerPodName == "" {
			continue
		}
		redis.Finalizers = append(redis.Finalizers, finalizerPodName)
		updateFlag = true
	}
	//更新Kind Redis状态
	if updateFlag {
		err := r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}
func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myapp1v1.Redis) error {
	/*
		//finalizers > num  可能出现，删除差值部分
		//finalizers = num 可能出现，删除全部
		//finalizers < num 不可能出现
	*/
	var deletedPodNames []string

	//删除finalizers切片的最后的指定元素
	position := redis.Spec.Num
	if (len(redis.Finalizers) - redis.Spec.Num) > 0 {
		deletedPodNames = redis.Finalizers[position:]
		redis.Finalizers = redis.Finalizers[:position]
	} else {
		deletedPodNames = redis.Finalizers[:position]
		redis.Finalizers = []string{}
	}

	fmt.Println("deletedPodNames", deletedPodNames)
	fmt.Println("redis.Finalizers", redis.Finalizers)
	for _, finalizer := range deletedPodNames {
		//删除pod
		err := r.Client.Delete(ctx, &v1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      finalizer,
				Namespace: redis.Namespace,
			},
		})
		if err != nil {
			return err
		}
	}
	redis.Finalizers = []string{}
	return r.Client.Update(ctx, redis)
}
```
## make run 开始测试
控制台执行make run......控制台2 执行apply -f test/redis.yaml创建3个副本pod
```
apiVersion: myapp1.zhangpeng.com/v1
kind: Redis
metadata:
  name: zhangpeng3
spec:
  port: 6379
  num: 3
  image: redis:latest
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077457268-18927f30-e8fb-4355-9cff-a72b9aca0e8c.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=981&id=uecf19131&margin=%5Bobject%20Object%5D&name=image.png&originHeight=883&originWidth=1437&originalType=binary&ratio=1&rotation=0&showTitle=false&size=175533&status=done&style=none&taskId=ub50273e7-9385-41c4-8ace-f75a197d633&title=&width=1596.6667089638897)
修改yaml副本数量为2,缩容pod apply yaml文件，缩容成功，删除了zhangpeng3-2pod！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077482968-4f8e728b-4f5c-4688-a057-445d133ee5d6.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=982&id=u0ab4dcf5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=884&originWidth=1274&originalType=binary&ratio=1&rotation=0&showTitle=false&size=167021&status=done&style=none&taskId=u28ca0e67-7378-40ef-b4b8-2f793d4afd5&title=&width=1415.5555930549726)
but！问题来了我接着想扩容副本数量到3.....不能正常操作了......不能生成新的pod
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077510485-176b9177-0380-40c4-bd2b-82ab08939a05.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=988&id=uf2ddd1ac&margin=%5Bobject%20Object%5D&name=image.png&originHeight=889&originWidth=1517&originalType=binary&ratio=1&rotation=0&showTitle=false&size=176694&status=done&style=none&taskId=uf07e0a93-0450-4054-8e71-ffe9fbf0c44&title=&width=1685.5556002075302)
观查一下make run报错......
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077527307-2466a01c-cafe-44f0-a98e-1a7a798ebfb4.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=987&id=u1ace5be1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=888&originWidth=1710&originalType=binary&ratio=1&rotation=0&showTitle=false&size=201162&status=done&style=none&taskId=u73d52e28-6fd0-4e29-b24d-716d3d3a3bf&title=&width=1900.0000503328124)
删除这几个资源重新来一遍，仔细观察一下步骤看看缺少了什么：
删除资源的时候明显发现绑定到pod跟Redis的资源没有绑定了？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077820720-6f9a9a4a-478e-4b93-ad72-0cbb4801c22b.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=350&id=u5edcfc72&margin=%5Bobject%20Object%5D&name=image.png&originHeight=315&originWidth=891&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64693&status=done&style=none&taskId=u61e4063e-1788-413c-a8d3-698abbb199a&title=&width=990.0000262260444)
apply yaml 副本数量为2, Finalizers中数据：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657077929006-56c565d5-7651-4cc0-b189-255370b38761.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=864&id=u5663b0c1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=778&originWidth=1319&originalType=binary&ratio=1&rotation=0&showTitle=false&size=125316&status=done&style=none&taskId=ub240230d-e967-43aa-861b-5c54fdbcf45&title=&width=1465.5555943795202)
修改副本数为2......这 标签就没有了？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657078031752-4b1c106e-b98f-4379-a7e4-5c7c12ee180d.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=953&id=u9ed67e52&margin=%5Bobject%20Object%5D&name=image.png&originHeight=858&originWidth=1225&originalType=binary&ratio=1&rotation=0&showTitle=false&size=141612&status=done&style=none&taskId=ucdd4348b-dccd-4ffb-9389-860dc1d7118&title=&width=1361.111147168243)
继续修改副本数为3....恩问题应该还是出在绑定finalizers这里......
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657078081393-08056d62-0ee9-413a-ace2-3498f83cd8b4.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=947&id=u0788ea6e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=852&originWidth=1138&originalType=binary&ratio=1&rotation=0&showTitle=false&size=128213&status=done&style=none&taskId=uba7b2aa8-56ee-4e29-a7bf-505d7670053&title=&width=1264.444477940784)
# 最终代码如下：
以下代码摘抄自[https://github.com/ls-2018/k8s-kustomize](https://github.com/ls-2018/k8s-kustomize)
## 实现扩容缩容删除恢复：
helper/redis_help.redis
```
package helper

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	v1 "kube-oprator1/api/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

func GetRedisPodNames(redisConfig *v1.Redis) []string {
	podNames := make([]string, redisConfig.Spec.Num)
	fmt.Printf("%+v", redisConfig)
	for i := 0; i < redisConfig.Spec.Num; i++ {
		podNames[i] = fmt.Sprintf("%s-%d", redisConfig.Name, i)
	}

	fmt.Println("PodNames: ", podNames)
	return podNames
}

//  判断 redis  pod 是否能获取
func IsExistPod(podName string, redis *v1.Redis, client client.Client) bool {
	err := client.Get(context.Background(), types.NamespacedName{
		Namespace: redis.Namespace,
		Name:      podName,
	},
		&corev1.Pod{},
	)

	if err != nil {
		return false
	}
	return true
}

func IsExistInFinalizers(podName string, redis *v1.Redis) bool {
	for _, fPodName := range redis.Finalizers {
		if podName == fPodName {
			return true

		}
	}
	return false
}

func CreateRedis(client client.Client, redisConfig *v1.Redis, podName string, schema *runtime.Scheme) (string, error) {
	if IsExistPod(podName, redisConfig, client) {
		return podName, nil
	}

	newPod := &corev1.Pod{}
	newPod.Name = podName
	newPod.Namespace = redisConfig.Namespace
	newPod.Spec.Containers =append(newPod.Spec.Containers, []corev1.Container{
			Name:            redisConfig.Name,
			Image:           redisConfig.Spec.Image,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Ports: []corev1.ContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		})

	//  set  owner  reference
	err := controllerutil.SetControllerReference(redisConfig, newPod, schema)
	if err != nil {
		return "", err
	}

return podName, client.Create(context.Background(), newPod)
}
```
controllers/redis_controller.go
```
/*
Copyright 2022 zhang peng.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/tools/record"
	"k8s.io/client-go/util/workqueue"
	myapp1v1 "kube-oprator1/api/v1"
	"kube-oprator1/helper"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"sigs.k8s.io/controller-runtime/pkg/source"
)

// RedisReconciler reconciles a Redis object
type RedisReconciler struct {
	client.Client
	Scheme      *runtime.Scheme
	EventRecord record.EventRecorder
}

//+kubebuilder:rbac:groups=myapp1.zhangpeng.com,resources=redis,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=myapp1.zhangpeng.com,resources=redis/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=myapp1.zhangpeng.com,resources=redis/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Redis object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.11.2/pkg/reconcile
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)
	redis := &myapp1v1.Redis{}
	// 删除后会往这发一个请求,只有NamespacedName数据，别的都没有
	if err := r.Get(ctx, req.NamespacedName, redis); err != nil {
		return ctrl.Result{}, nil
	}
	// 正在删除
	if !redis.DeletionTimestamp.IsZero() {
		return ctrl.Result{}, r.clearPods(context.Background(), redis)
	}
	// TODO pod 个数变动
	podNames := helper.GetRedisPodNames(redis)
	var err error
	if redis.Spec.Num > len(redis.Finalizers) {
		err = r.UpPods(ctx, podNames, redis)
		if err == nil {
			fmt.Printf("%d", redis.Spec.Num)
			//r.EventRecord.Event(redis, corev1.EventTypeNormal, "UpPods", fmt.Sprintf("%d", redis.Spec.Num))
		} else {
			//			r.EventRecord.Event(redis, corev1.EventTypeWarning, "DownPods", fmt.Sprintf("%d", redis.Spec.Num))
		}
	} else if redis.Spec.Num < len(redis.Finalizers) {
		err = r.DownPods(ctx, podNames, redis)
		if err == nil {
			//			r.EventRecord.Event(redis, corev1.EventTypeNormal, "DownPods", fmt.Sprintf("%d", redis.Spec.Num))
		} else {
			//		r.EventRecord.Event(redis, corev1.EventTypeWarning, "DownPods", fmt.Sprintf("%d", redis.Spec.Num))
		}
		redis.Status.RedisNum = len(redis.Finalizers)
	} else {
		for _, podName := range redis.Finalizers {
			if helper.IsExistPod(podName, redis, r.Client) {
				continue
			} else {
				//	 重建此pod
				err = r.UpPods(ctx, []string{podName}, redis)
				if err != nil {
					return ctrl.Result{}, err
				}
			}
		}
	}
	r.Status().Update(ctx, redis)

	return ctrl.Result{}, err
}

// SetupWithManager sets up the controller with the Manager.
func (r *RedisReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&myapp1v1.Redis{}).
		Watches(&source.Kind{Type: &corev1.Pod{}}, handler.Funcs{
			CreateFunc:  nil,
			UpdateFunc:  nil,
			DeleteFunc:  r.podDeleteHandler,
			GenericFunc: nil,
		}).
		Complete(r)
}

// 对于用户主动 删除的pod 需要重新创建
func (r *RedisReconciler) podDeleteHandler(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface) {
	fmt.Printf(`######################
%s
######################
`, event.Object.GetName())
	for _, ref := range event.Object.GetOwnerReferences() {
		if ref.Kind == r.kind() && ref.APIVersion == r.apiVersion() {
			//触发 Reconcile
			limitingInterface.Add(reconcile.Request{
				NamespacedName: types.NamespacedName{
					Namespace: event.Object.GetNamespace(),
					Name:      ref.Name,
				},
			})
		}
	}
}

func (r *RedisReconciler) kind() string {
	return "Redis"
}

func (r *RedisReconciler) apiVersion() string {
	return "myapp1.zhangpeng.com/v1"
}

func (r *RedisReconciler) clearPods(ctx context.Context, redis *myapp1v1.Redis) error {
	for _, podName := range redis.Finalizers {
		err := r.Client.Delete(ctx, &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      podName,
				Namespace: redis.Namespace,
			},
		})
		//TODO 如果pod已经被删除、
		if err != nil {
			return err
		}
	}
	redis.Finalizers = []string{}
	return r.Client.Update(ctx, redis)
}
func (r *RedisReconciler) DownPods(ctx context.Context, podNames []string, redis *myapp1v1.Redis) error {
	for i := len(redis.Finalizers) - 1; i >= len(podNames); i-- {
		if !helper.IsExistPod(redis.Finalizers[i], redis, r.Client) {
			continue
		}
		err := r.Client.Delete(ctx, &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      redis.Finalizers[i],
				Namespace: redis.Namespace,
			},
		})
		if err != nil {
			return err
		}
	}
	redis.Finalizers = append(redis.Finalizers[:0], redis.Finalizers[:len(podNames)]...)
	return r.Client.Update(ctx, redis)
}
func (r *RedisReconciler) UpPods(ctx context.Context, podNames []string, redis *myapp1v1.Redis) error {
	for _, podName := range podNames {
		podName, err := helper.CreateRedis(r.Client, redis, podName, r.Scheme)
		if err != nil {
			return err
		}
		if controllerutil.ContainsFinalizer(redis, podName) {
			continue
		}
		redis.Finalizers = append(redis.Finalizers, podName)
	}
	err := r.Client.Update(ctx, redis)
	return err
}

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657160745630-ac0d9a78-fa70-4b34-9d85-88b940513311.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=956&id=u5fab94d4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=860&originWidth=1354&originalType=binary&ratio=1&rotation=0&showTitle=false&size=161112&status=done&style=none&taskId=ubee97b42-b3c6-4f94-95c7-585a9250333&title=&width=1504.444484298613)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657160777908-f1883df6-3e3c-49ff-a160-7b1cae9b7516.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=994&id=u3f288cac&margin=%5Bobject%20Object%5D&name=image.png&originHeight=895&originWidth=1281&originalType=binary&ratio=1&rotation=0&showTitle=false&size=165298&status=done&style=none&taskId=u439f3632-ef10-4cd5-ab83-f52f77a7ae4&title=&width=1423.333371038791)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657160801315-72c02254-af39-461c-814f-b2c346593cdd.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=971&id=ub24a5499&margin=%5Bobject%20Object%5D&name=image.png&originHeight=874&originWidth=1219&originalType=binary&ratio=1&rotation=0&showTitle=false&size=152588&status=done&style=none&taskId=u9da5785a-7587-4975-bab9-928388379ad&title=&width=1354.4444803249698)
删除zhangpeng3  Redis

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657160828977-c0a00d9e-8eb9-4085-a518-28c74adace1d.png#clientId=u337531bd-212e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=974&id=u45e0fc5e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=877&originWidth=1424&originalType=binary&ratio=1&rotation=0&showTitle=false&size=168798&status=done&style=none&taskId=u5beae557-e1f4-4863-adcd-a9d395b21f5&title=&width=1582.2222641367982)
基本功能算是完成了，测试一下Pod重建：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657509732900-850283d5-4d06-45e1-9270-4aaa6430da6d.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=680&id=u269d7011&margin=%5Bobject%20Object%5D&name=image.png&originHeight=612&originWidth=1396&originalType=binary&ratio=1&rotation=0&showTitle=false&size=116607&status=done&style=none&taskId=u7e268c00-aa00-4525-b001-02490921fcc&title=&width=1551.1111522015242)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657509772572-3ce36171-f174-4a7c-8047-33a9f127b2d4.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=701&id=u315ee6a1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=631&originWidth=1713&originalType=binary&ratio=1&rotation=0&showTitle=false&size=240681&status=done&style=none&taskId=uea4f5bac-2a93-498d-ba3e-50442c0b4d2&title=&width=1903.333383754449)
## 其他的
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657513367469-cb7690cf-689b-4f7e-ad91-ff7026315adb.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=257&id=uba240977&margin=%5Bobject%20Object%5D&name=image.png&originHeight=231&originWidth=821&originalType=binary&ratio=1&rotation=0&showTitle=false&size=40757&status=done&style=none&taskId=u5fa6d137-1abb-4fe1-bee8-c71ed1c0e94&title=&width=912.2222463878591)
api/v1/redis_types.go
```
// RedisStatus defines the observed state of Redis
type RedisStatus struct {
	RedisNum int `json:"num,omitempty"`
}

//下面两个注释对应的是 kubectl get Redis 看到的状态
//+kubebuilder:printcolumn:JSONPath=".status.num",name=NUM,type=integer
//+kubebuilder:printcolumn:JSONPath=".metadata.creationTimestamp",name=AGE,type=date
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657513896029-f51d3b13-e691-477f-944f-cd109783f8eb.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=916&id=u27072f93&margin=%5Bobject%20Object%5D&name=image.png&originHeight=824&originWidth=1455&originalType=binary&ratio=1&rotation=0&showTitle=false&size=213625&status=done&style=none&taskId=u26dccf92-fc56-4b74-8239-7e040cd2275&title=&width=1616.6667094937088)
```
[zhangpeng@zhangpeng kube-oprator1]$ make install
[zhangpeng@zhangpeng kube-oprator1]$ make run
```
看一下config/crd/bases/myapp1.zhangpeng.com_redis.yaml
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657514112510-8d7bbaf5-e228-47c9-8173-57750245169e.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=872&id=u2eda6cbe&margin=%5Bobject%20Object%5D&name=image.png&originHeight=785&originWidth=1703&originalType=binary&ratio=1&rotation=0&showTitle=false&size=194902&status=done&style=none&taskId=uf052a987-8147-4eff-be23-2c81a319d33&title=&width=1892.222272348994)
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657514133012-1bd0a4e0-5849-474a-8c9d-71f7fdcc49bd.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=323&id=u2a6715b4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=291&originWidth=810&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55613&status=done&style=none&taskId=u2b1dfcd5-b414-492c-bf0b-04be5166b59&title=&width=900.0000238418586)
## 增加Event事件支持
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get Redis
NAME         NUM   AGE
zhangpeng3   2     82m
[zhangpeng@zhangpeng kube-oprator1]$ kubectl describe Redis zhangpeng3
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657520700375-834dac57-828e-4fe6-b9e3-4121e28d8ca0.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=714&id=u49947664&margin=%5Bobject%20Object%5D&name=image.png&originHeight=643&originWidth=1006&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66065&status=done&style=none&taskId=ua1d7c469-f8f0-480a-a13d-256aaa20815&title=&width=1117.7778073887773)
观察我redis_controller.go代码的时候发现我注释了这些代码......这些其实就是event的输出，不知道怎么会事情就空指针报错了 ，就注释掉了！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657522382840-e5be5887-162e-4c23-9cb9-2a531be2a673.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=850&id=u0e0ef1f6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=765&originWidth=1567&originalType=binary&ratio=1&rotation=0&showTitle=false&size=215671&status=done&style=none&taskId=u89268c1b-3a97-4496-af30-8a0a6fadade&title=&width=1741.1111572348054)
什么原因呢？参照：[https://github.com/767829413/learn-to-apply/blob/3621065003918070635880c3348f1d72bf6ead88/docs/Kubernetes/kubernetes-secondary-development.md](https://github.com/767829413/learn-to-apply/blob/3621065003918070635880c3348f1d72bf6ead88/docs/Kubernetes/kubernetes-secondary-development.md)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657525643561-f32802cf-4bc8-43f9-bfae-d5217bfd6b01.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=607&id=u778a2b87&margin=%5Bobject%20Object%5D&name=image.png&originHeight=546&originWidth=1499&originalType=binary&ratio=1&rotation=0&showTitle=false&size=121030&status=done&style=none&taskId=u25fce07b-594d-45ad-8012-bad467fb7c6&title=&width=1665.5555996777111)
修改main.go
```
	if err = (&controllers.RedisReconciler{
		Client:      mgr.GetClient(),
		Scheme:      mgr.GetScheme(),
		EventRecord: mgr.GetEventRecorderFor("myapp1.zhangpeng.com"), //名称任意
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "Redis")
		os.Exit(1)
	}
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657525667986-5efca21a-d7cb-41a9-b079-9c71f9f0cc1e.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=590&id=ua0100283&margin=%5Bobject%20Object%5D&name=image.png&originHeight=531&originWidth=1485&originalType=binary&ratio=1&rotation=0&showTitle=false&size=145693&status=done&style=none&taskId=uc76d2cce-8872-450d-90f8-147f0c47706&title=&width=1650.000043710074)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657525725796-04d38375-bff2-4414-9c35-b6093fa13788.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=937&id=ue23baed1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=843&originWidth=1444&originalType=binary&ratio=1&rotation=0&showTitle=false&size=132260&status=done&style=none&taskId=u2b7954a1-f6cb-432c-b327-126689b972d&title=&width=1604.4444869477084)
注：刚才修改副本为2了，又修改成4进行测试的！
# 最终打包发布：
## webhook
打包发布第一步骤如果开启webhook
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657525826668-9a4073f6-ae75-418e-9173-1930d17e130c.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=878&id=u7e5f078b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=790&originWidth=1637&originalType=binary&ratio=1&rotation=0&showTitle=false&size=218066&status=done&style=none&taskId=u6e298385-5e26-4e31-abda-af173756f00&title=&width=1818.8889370729908)
## IMG build
```
[zhangpeng@zhangpeng kube-oprator1]$ make install
[zhangpeng@zhangpeng kube-oprator1]$ make docker-build docker-push IMG=ccr.ccs.tencentyun.com/layatools/zpredis:v5
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657525999165-c1cfcd88-ef85-4640-a860-17ac3efd99ca.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=621&id=u1229690f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=559&originWidth=1542&originalType=binary&ratio=1&rotation=0&showTitle=false&size=138500&status=done&style=none&taskId=ud962fd71-96ca-4efc-b370-ed06d61628d&title=&width=1713.3333787211677)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657526435888-c8cc1d87-6862-4670-b930-655d35e46054.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=508&id=ua6af0417&margin=%5Bobject%20Object%5D&name=image.png&originHeight=457&originWidth=1169&originalType=binary&ratio=1&rotation=0&showTitle=false&size=86464&status=done&style=none&taskId=u4bdb5b45-420f-49be-929f-f87bed22ddb&title=&width=1298.8889232976946)
Dockerfile中增加一下配置：

```
COPY helper/ helper/
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657526606632-ba0380a6-6365-49fc-8f69-e39099168a5b.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=583&id=u5e2412c6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=525&originWidth=1732&originalType=binary&ratio=1&rotation=0&showTitle=false&size=120851&status=done&style=none&taskId=u8a0b8c77-c36a-41dd-b18d-27c3edf1277&title=&width=1924.4444954248136)
继续执行构建上传镜像：

```
[zhangpeng@zhangpeng kube-oprator1]$ make docker-build docker-push IMG=ccr.ccs.tencentyun.com/layatools/zpredis:v5

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657526935179-7a5dd6d1-6289-4718-a9ae-9d58db83680a.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=620&id=ua13eb192&margin=%5Bobject%20Object%5D&name=image.png&originHeight=558&originWidth=1452&originalType=binary&ratio=1&rotation=0&showTitle=false&size=110384&status=done&style=none&taskId=ue63e9934-98c2-494b-a34d-84954bbe7ea&title=&width=1613.3333760720723)
## 发布
正常流程：
```
[zhangpeng@zhangpeng kube-oprator1]$ make deploy IMG=ccr.ccs.tencentyun.com/layatools/zpredis:v5
```
由于 我的环境问题还要拆解一下命令：

```
[zhangpeng@zhangpeng kube-oprator1]$ cd config/manager &&  kustomize edit set image controller=ccr.ccs.tencentyun.com/layatools/zpredis:v5
[zhangpeng@zhangpeng manager]$ cd ../../
[zhangpeng@zhangpeng kube-oprator1]$ kustomize build config/default | kubectl apply -f -
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527028490-59469644-839b-497a-a685-2f78192d0a40.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=672&id=u3df9cec1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=605&originWidth=1569&originalType=binary&ratio=1&rotation=0&showTitle=false&size=161181&status=done&style=none&taskId=u1287c769-ebfc-4d0a-8683-515f039b557&title=&width=1743.3333795158965)
## 权限
```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods -n kube-oprator1-system 
NAME                                                READY   STATUS    RESTARTS   AGE
kube-oprator1-controller-manager-84c7dcf9fc-xxnk8   2/2     Running   
[zhangpeng@zhangpeng kube-oprator1]$ kubectl logs -f kube-oprator1-controller-manager-84c7dcf9fc-xxnk8 -n  kube-oprator1-system 
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527692801-8ed0053a-7f73-4f08-97f2-986e663575b0.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=570&id=u10ec4c13&margin=%5Bobject%20Object%5D&name=image.png&originHeight=513&originWidth=1663&originalType=binary&ratio=1&rotation=0&showTitle=false&size=200360&status=done&style=none&taskId=u5420d014-5be7-4393-8a54-670bf20df9c&title=&width=1847.7778267271738)
恩 权限不够 临时搞一个clusterrolebinding ：

```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get sa kube-oprator1-controller-manager  -n kube-oprator1-system -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"kube-oprator1-controller-manager","namespace":"kube-oprator1-system"}}
  creationTimestamp: "2022-06-30T10:53:04Z"
  name: kube-oprator1-controller-manager
  namespace: kube-oprator1-system
  resourceVersion: "1917"
  uid: 46634b00-0c32-4862-8afb-17bb9d2181d0
secrets:
- name: kube-oprator1-controller-manager-token-x2c9f
[zhangpeng@zhangpeng kube-oprator1]$ kubectl create clusterrolebinding kube-oprator1-system --clusterrole cluster-admin --serviceaccount=kube-oprator1-system:kube-oprator1-controller-manager
clusterrolebinding.rbac.authorization.k8s.io/kube-oprator1-system created

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527796555-af54cda7-372c-405f-9319-191af6611eb4.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=631&id=ua5defd1d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=568&originWidth=1664&originalType=binary&ratio=1&rotation=0&showTitle=false&size=117649&status=done&style=none&taskId=uc4daf3f2-3e86-4b9f-8192-d61f827f5fb&title=&width=1848.8889378677193)
delete pod等其启动：

```
[zhangpeng@zhangpeng kube-oprator1]$ kubectl delete pods kube-oprator1-controller-manager-84c7dcf9fc-xxnk8 -n  kube-oprator1-system 
pod "kube-oprator1-controller-manager-84c7dcf9fc-xxnk8" deleted
[zhangpeng@zhangpeng kube-oprator1]$ kubectl get pods -n kube-oprator1-system 
NAME                                                READY   STATUS    RESTARTS   AGE
kube-oprator1-controller-manager-84c7dcf9fc-fbzsl   1/2     Running   0          4s
[zhangpeng@zhangpeng kube-oprator1]$ kubectl logs -f kube-oprator1-controller-manager-84c7dcf9fc-fbzsl -n  kube-oprator1-system 
```
修改副本3为2：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527900073-f12b50cb-7ee4-4db9-847f-d708cc337986.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=821&id=u0ea3cf6c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=739&originWidth=1718&originalType=binary&ratio=1&rotation=0&showTitle=false&size=221072&status=done&style=none&taskId=ua314cf71-8f8c-4b79-a329-9b42b8c5295&title=&width=1908.8889394571765)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527960537-c993aa1b-b83d-4243-a780-7ad494f4acd0.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=733&id=u41f48e31&margin=%5Bobject%20Object%5D&name=image.png&originHeight=660&originWidth=1694&originalType=binary&ratio=1&rotation=0&showTitle=false&size=236163&status=done&style=none&taskId=u4a012872-4810-4223-ab0d-913a8dff3ca&title=&width=1882.2222720840844)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657527991780-87b60bb6-9b95-45f5-92eb-3d7326608035.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=642&id=u7a7f8661&margin=%5Bobject%20Object%5D&name=image.png&originHeight=578&originWidth=1080&originalType=binary&ratio=1&rotation=0&showTitle=false&size=92715&status=done&style=none&taskId=u2ddc6f0d-e607-4a91-9dc0-3552e2f42be&title=&width=1200.0000317891447)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657528004252-cba98573-a35d-4bcd-b813-ac396cd08278.png#clientId=uc659eb64-2610-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=559&id=u94e1756b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=503&originWidth=916&originalType=binary&ratio=1&rotation=0&showTitle=false&size=58175&status=done&style=none&taskId=u7bd538b4-aedf-4931-b9aa-a659e5ba837&title=&width=1017.777804739682)
基本完成，不过深究一下,pod删除了重建是不是也可以增加一下记录？
# 总结：
参考一下github地址或者文章：
[https://github.com/ls-2018/k8s-kustomize](https://github.com/ls-2018/k8s-kustomize/blob/main/redis/helper/redis_helper.go)
关于Event修改main.go参照：[https://github.com/767829413/learn-to-apply/blob/3621065003918070635880c3348f1d72bf6ead88/docs/Kubernetes/kubernetes-secondary-development.md](https://github.com/767829413/learn-to-apply/blob/3621065003918070635880c3348f1d72bf6ead88/docs/Kubernetes/kubernetes-secondary-development.md)
文章可以参考：[https://www.cnblogs.com/cosmos-wong/p/15894689.html](https://www.cnblogs.com/cosmos-wong/p/15894689.html)
[https://podsbook.com/posts/kubernetes/operator/#%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5](https://podsbook.com/posts/kubernetes/operator/#%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5)这个文章也不错！
finalizers参照：[https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/)
至于关于redis  operator的文章以及github地址来说除了[https://podsbook.com/posts/kubernetes/operator](https://podsbook.com/posts/kubernetes/operator) 还有finalizers官方文档，都应该是沈叔的课程[k8s基础速学3:Operator、Prometheus、日志收集](https://www.jtthink.com/course/154)中的内容来的！
接下来准备写一下自己的operator......
