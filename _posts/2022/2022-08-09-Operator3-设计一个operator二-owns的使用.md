---
layout: post
title: 2022-08-09-Operator3-设计一个operator二-owns的使用
date: 2022-08-09 2:00:00
category: kubernetes
tags: kubernetes operator
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
上一节（[Operator3-设计一个operator](https://www.yuque.com/duiniwukenaihe/hg6ymd/cgvon5)）做完发现一个问题  我创建了jan 应用jan-sample，子资源包括deployment，service.ingress,pod(其中pod是deployment管理的) 
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659528000127-15866844-c226-40bb-b0d3-fb8de36962f9.png#clientId=u8920b99d-e7b3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=152&id=u1308682e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=137&originWidth=987&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28867&status=done&style=none&taskId=uf1c5964c-59fd-486d-803d-4df55f0ca1e&title=&width=1096.6666957184127)
手动删除Pod.由于Deployment rc控制器。Pod资源可以自动重建。但是我删除deployment能不能自动重建呢？正常的deployment service ingress子资源的生命周期，我应该是靠jan应用去维系的，试一试：
```
[zhangpeng@zhangpeng jan]$ kubectl delete deployment jan-sample
deployment.apps "jan-sample" deleted
[zhangpeng@zhangpeng jan]$ kubectl get deployment
No resources found in default namespace.

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659528168440-3608a6d2-5623-4179-952e-64f49f46192e.png#clientId=u8920b99d-e7b3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=210&id=u5af22a9d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=189&originWidth=1222&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42096&status=done&style=none&taskId=u9cb304e4-3aee-4cb1-9a93-90ded5dd873&title=&width=1357.7778137466064)
到这里才发现没有考虑周全......删除deployment资源并不能重建，正常创建应用应该要考虑一下jan资源下面资源的重建.搜了一下别人写的operator貌似的可以加一下**Owns**，尝试一下！
# Deployment Ingress Service关于Owns的使用
## Deployment
```
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}
```
### Deployment delete尝试
make run develop-operator项目，并尝试delete deployment jan-sample查看是否重建：
```
[zhangpeng@zhangpeng develop-operator]$ kubectl get Jan
[zhangpeng@zhangpeng develop-operator]$ kubectl get all
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659530912074-8ceac1b7-0334-46b0-b0c1-ee8ef0d5ed2f.png#clientId=u8920b99d-e7b3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=419&id=u221ec4da&margin=%5Bobject%20Object%5D&name=image.png&originHeight=377&originWidth=1021&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66141&status=done&style=none&taskId=uf1b61098-16a8-4e51-968f-4976d2a2e7f&title=&width=1134.4444744969599)

```
[zhangpeng@zhangpeng develop-operator]$ kubectl delete deployment jan-sample
[zhangpeng@zhangpeng develop-operator]$ kubectl get deployment
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659530950179-e44d27bb-0a0d-44cb-b255-51350b2189d9.png#clientId=u8920b99d-e7b3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=370&id=u612daa44&margin=%5Bobject%20Object%5D&name=image.png&originHeight=333&originWidth=1022&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64143&status=done&style=none&taskId=u4afa87f9-5c65-4753-bd78-01fa191b8be&title=&width=1135.5555856375054)
恩发现deployment应用可以自动重建了！

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659531464374-e00c465b-c5a7-4be9-bbaa-e3de24356a05.png#clientId=u8920b99d-e7b3-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=679&id=uab1469fa&margin=%5Bobject%20Object%5D&name=image.png&originHeight=611&originWidth=1316&originalType=binary&ratio=1&rotation=0&showTitle=false&size=127261&status=done&style=none&taskId=uf8d552a0-16b8-4034-bfeb-fa7e02ccc46&title=&width=1462.2222609578837)
## Ingress and Service资源
### 简单添加一下Owns？
but其他资源是否可以呢？是不是也偷懒一下添加**Owns**？
```
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Owns(&v1.Ingress{}).
		Complete(r)
}
```
尝试了一下不生效的，但是这种方式思路是对的至于为什么不生效呢？
deploy声明了&appv1.Deployment,但是service,ingress是没有创建变量声明的！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1659953527109-e2868b4b-0611-4ccb-9791-e3d280a195f9.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=226&id=ue7d9e6bf&margin=%5Bobject%20Object%5D&name=image.png&originHeight=203&originWidth=646&originalType=binary&ratio=1&rotation=0&showTitle=false&size=32036&status=done&style=none&taskId=ua8c9aae1-9c45-4268-9ee8-fbdb33d2da5&title=&width=717.7777967923959)
### 拆分改造代码
继续改造Jan operator使其支持service ingress子资源的误删除创建：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660011155926-870d31ca-f0e5-4489-8450-614bf73545ca.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=598&id=u913bf42d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=538&originWidth=699&originalType=binary&ratio=1&rotation=0&showTitle=false&size=86422&status=done&style=none&taskId=u55ee35eb-1241-48f2-b6f9-54a4144effd&title=&width=776.6666872413075)
把这边拆分一下？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660035140956-da721743-fd26-48ec-8c32-b62b3362b579.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=560&id=ud2c3bfbf&margin=%5Bobject%20Object%5D&name=image.png&originHeight=504&originWidth=1092&originalType=binary&ratio=1&rotation=0&showTitle=false&size=84034&status=done&style=none&taskId=u33c622c8-ba40-4700-925b-98dc018c140&title=&width=1213.3333654756907)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660035164638-5804a65e-df73-4154-bf98-d179e1adb969.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=549&id=uf154cae7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=494&originWidth=1078&originalType=binary&ratio=1&rotation=0&showTitle=false&size=78945&status=done&style=none&taskId=u689f6efe-fd8f-4af9-bc2d-bb712b4723f&title=&width=1197.7778095080537)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660035188517-b9b82d22-9e50-4ecf-a075-d2bed326d85e.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=557&id=u6da450ce&margin=%5Bobject%20Object%5D&name=image.png&originHeight=501&originWidth=1152&originalType=binary&ratio=1&rotation=0&showTitle=false&size=79337&status=done&style=none&taskId=u08d562a7-81f9-4ff3-8915-3bef4e045ec&title=&width=1280.000033908421)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660035272984-cfaae0e3-d954-4ca8-967b-497cd01d9e54.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=343&id=u76002480&margin=%5Bobject%20Object%5D&name=image.png&originHeight=309&originWidth=810&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39839&status=done&style=none&taskId=u2eca81fc-588d-47a0-9e26-63296312b1f&title=&width=900.0000238418586)
jan_controller.go
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

package jan

import (
	"context"
	"encoding/json"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/api/networking/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"reflect"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	janv1 "develop-operator/apis/jan/v1"
)

// JanReconciler reconciles a Jan object
type JanReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=networking,resources=ingresses,verbs=get;list;watch;create;update;patch;delete

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Jan object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.11.2/pkg/reconcile
func (r *JanReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	defer utilruntime.HandleCrash()
	_ = log.FromContext(ctx)
	instance := &janv1.Jan{}
	err := r.Client.Get(context.TODO(), req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}
	if instance.DeletionTimestamp != nil {
		return reconcile.Result{}, err
	}

	// 如果不存在，则创建关联资源
	// 如果存在，判断是否需要更新
	//   如果需要更新，则直接更新
	//   如果不需要更新，则正常返回

	deploy := &appsv1.Deployment{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		// 创建关联资源
		// 1. 创建 Deploy
		deploy := NewJan(instance)
		if err := r.Client.Create(context.TODO(), deploy); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(instance.Spec)

		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	Service := &corev1.Service{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, Service); err != nil && errors.IsNotFound(err) {

		// 2. 创建 Service
		service := NewService(instance)
		if err := r.Client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(service.Spec)

		if service.Annotations != nil {
			service.Annotations["spec"] = string(data)
		} else {
			service.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), service); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	Ingress := &v1.Ingress{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, Ingress); err != nil && errors.IsNotFound(err) {

		// 2. 创建 Service
		ingress := NewIngress(instance)
		if err := r.Client.Create(context.TODO(), ingress); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(ingress.Spec)

		if ingress.Annotations != nil {
			ingress.Annotations["spec"] = string(data)
		} else {
			ingress.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), ingress); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	oldspec := janv1.JanSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), &oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
		data, _ := json.Marshal(instance.Spec)

		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		// 更新关联资源
		newDeploy := NewJan(instance)
		oldDeploy := &appsv1.Deployment{}
		if err := r.Client.Get(context.TODO(), req.NamespacedName, oldDeploy); err != nil {
			return reconcile.Result{}, err
		}
		oldDeploy.Spec = newDeploy.Spec
		if err := r.Client.Update(context.TODO(), oldDeploy); err != nil {
			return reconcile.Result{}, err
		}

		newService := NewService(instance)
		oldService := &corev1.Service{}
		if err := r.Client.Get(context.TODO(), req.NamespacedName, oldService); err != nil {
			return reconcile.Result{}, err
		}
		oldService.Spec = newService.Spec
		if err := r.Client.Update(context.TODO(), oldService); err != nil {
			return reconcile.Result{}, err
		}
		return reconcile.Result{}, nil
	}
	newStatus := janv1.JanStatus{
		Replicas:      *instance.Spec.Replicas,
		ReadyReplicas: instance.Status.Replicas,
	}

	if newStatus.Replicas == newStatus.ReadyReplicas {
		newStatus.Phase = janv1.Running
	} else {
		newStatus.Phase = janv1.NotReady
	}
	if !reflect.DeepEqual(instance.Status, newStatus) {
		instance.Status = newStatus
		log.FromContext(ctx).Info("update game status", "name", instance.Name)
		err = r.Client.Status().Update(ctx, instance)
		if err != nil {
			return reconcile.Result{}, err
		}
	}
	return reconcile.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Owns(&v1.Ingress{}).
		Complete(r)
}

```

make run尝试一下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036063175-3ee7e4ef-60b0-44c8-affa-c33b3d3046a4.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=544&id=ua19210a3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=490&originWidth=1539&originalType=binary&ratio=1&rotation=0&showTitle=false&size=162340&status=done&style=none&taskId=udc74dfa4-f641-416e-86e9-d829b0b03f9&title=&width=1710.0000452995312)
注意：make run之前默认删除jan应用！
```
[zhangpeng@zhangpeng develop-operator]$ kubectl delete svc jan-sample
[zhangpeng@zhangpeng develop-operator]$ kubectl get svc
```
en service的自动恢复生效了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036097535-feea4439-d540-4934-94af-31857f3aa1a5.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=548&id=u3dd025ac&margin=%5Bobject%20Object%5D&name=image.png&originHeight=493&originWidth=1113&originalType=binary&ratio=1&rotation=0&showTitle=false&size=93613&status=done&style=none&taskId=u7efc0b9b-e948-4c16-9bc7-b027417fb40&title=&width=1236.6666994271463)
然后试一试ingress
```
[zhangpeng@zhangpeng develop-operator]$ kubectl get ingress
[zhangpeng@zhangpeng develop-operator]$ kubectl delete ingress jan-sample
[zhangpeng@zhangpeng develop-operator]$ kubectl get ingress

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036222840-80818540-fa2a-43d1-b9ef-0ee78363f501.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=247&id=u45177af3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=222&originWidth=1053&originalType=binary&ratio=1&rotation=0&showTitle=false&size=46076&status=done&style=none&taskId=ud45ab86c-cb63-4336-8118-de2056425a9&title=&width=1170.0000309944162)
继续发现问题:
en,我修改一下jan_v1_jan.yaml中host ww1.zhangpeng.com修改为ww11.zhangpeng.com,but ingress的相关信息没有及时更新啊？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036302109-91ef321d-7aaf-444e-abf3-3fa3e0a05310.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=903&id=u6ff19330&margin=%5Bobject%20Object%5D&name=image.png&originHeight=813&originWidth=1527&originalType=binary&ratio=1&rotation=0&showTitle=false&size=167880&status=done&style=none&taskId=ud3a24346-8f84-45f7-a0ba-58f4d9ad0d8&title=&width=1696.6667116129852)
继续模仿一下上面的service oldservice newservice新增 newIngress  oldIngress ：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036326647-c47b4e3c-658e-4076-b1f5-10bd85b16770.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=541&id=u01b83f4f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=487&originWidth=1186&originalType=binary&ratio=1&rotation=0&showTitle=false&size=93278&status=done&style=none&taskId=u1b2c5fd9-8062-4d42-9373-bc791fc0d9d&title=&width=1317.7778126869682)
重新make run
ingress相关信息得到了修改
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1660036450676-696614ad-9106-4495-97cb-f496a070a854.png#clientId=u70c9a5f9-fd6a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1010&id=uc6c8ef64&margin=%5Bobject%20Object%5D&name=image.png&originHeight=909&originWidth=1590&originalType=binary&ratio=1&rotation=0&showTitle=false&size=218706&status=done&style=none&taskId=u0ab84249-ca04-482e-b600-80ca6b7da3d&title=&width=1766.6667134673519)
## 最终代码：
jan_controller.go
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

package jan

import (
	"context"
	"encoding/json"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/api/networking/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"reflect"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	janv1 "develop-operator/apis/jan/v1"
)

// JanReconciler reconciles a Jan object
type JanReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=mar.zhangpeng.com,resources=jan/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=networking,resources=ingresses,verbs=get;list;watch;create;update;patch;delete

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Jan object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.11.2/pkg/reconcile
func (r *JanReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	defer utilruntime.HandleCrash()
	_ = log.FromContext(ctx)
	instance := &janv1.Jan{}
	err := r.Client.Get(context.TODO(), req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}
	if instance.DeletionTimestamp != nil {
		return reconcile.Result{}, err
	}

	// 如果不存在，则创建关联资源
	// 如果存在，判断是否需要更新
	//   如果需要更新，则直接更新
	//   如果不需要更新，则正常返回

	deploy := &appsv1.Deployment{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		// 创建关联资源
		// 1. 创建 Deploy
		deploy := NewJan(instance)
		if err := r.Client.Create(context.TODO(), deploy); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(instance.Spec)

		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	Service := &corev1.Service{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, Service); err != nil && errors.IsNotFound(err) {

		// 2. 创建 Service
		service := NewService(instance)
		if err := r.Client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(service.Spec)

		if service.Annotations != nil {
			service.Annotations["spec"] = string(data)
		} else {
			service.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), service); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	Ingress := &v1.Ingress{}

	if err := r.Client.Get(context.TODO(), req.NamespacedName, Ingress); err != nil && errors.IsNotFound(err) {

		// 2. 创建 Ingress
		ingress := NewIngress(instance)
		if err := r.Client.Create(context.TODO(), ingress); err != nil {
			return reconcile.Result{}, err
		}
		// 4. 关联 Annotations
		data, _ := json.Marshal(ingress.Spec)

		if ingress.Annotations != nil {
			ingress.Annotations["spec"] = string(data)
		} else {
			ingress.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), ingress); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}
	oldspec := janv1.JanSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), &oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
		data, _ := json.Marshal(instance.Spec)

		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}
		if err := r.Client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		// 更新关联资源
		newDeploy := NewJan(instance)
		oldDeploy := &appsv1.Deployment{}
		if err := r.Client.Get(context.TODO(), req.NamespacedName, oldDeploy); err != nil {
			return reconcile.Result{}, err
		}
		oldDeploy.Spec = newDeploy.Spec
		if err := r.Client.Update(context.TODO(), oldDeploy); err != nil {
			return reconcile.Result{}, err
		}

		newService := NewService(instance)
		oldService := &corev1.Service{}
		if err := r.Client.Get(context.TODO(), req.NamespacedName, oldService); err != nil {
			return reconcile.Result{}, err
		}
		oldService.Spec = newService.Spec
		if err := r.Client.Update(context.TODO(), oldService); err != nil {
			return reconcile.Result{}, err
		}
		newIngress := NewIngress(instance)
		oldIngress := &v1.Ingress{}
		if err := r.Client.Get(context.TODO(), req.NamespacedName, oldIngress); err != nil {
			return reconcile.Result{}, err
		}
		oldIngress.Spec = newIngress.Spec
		if err := r.Client.Update(context.TODO(), oldIngress); err != nil {
			return reconcile.Result{}, err
		}
		return reconcile.Result{}, nil
	}
	newStatus := janv1.JanStatus{
		Replicas:      *instance.Spec.Replicas,
		ReadyReplicas: instance.Status.Replicas,
	}

	if newStatus.Replicas == newStatus.ReadyReplicas {
		newStatus.Phase = janv1.Running
	} else {
		newStatus.Phase = janv1.NotReady
	}
	if !reflect.DeepEqual(instance.Status, newStatus) {
		instance.Status = newStatus
		log.FromContext(ctx).Info("update game status", "name", instance.Name)
		err = r.Client.Status().Update(ctx, instance)
		if err != nil {
			return reconcile.Result{}, err
		}
	}
	return reconcile.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Owns(&appsv1.Deployment{}).
		Owns(&corev1.Service{}).
		Owns(&v1.Ingress{}).
		Complete(r)
}

```
# 总结

1. owns的一般使用
1. 将 deployment service ingress或者其他资源作为operator应用的子资源，进行生命周期管理
1. 下一步想处理一下 make run 控制台的输出，输出一些有用的信息



