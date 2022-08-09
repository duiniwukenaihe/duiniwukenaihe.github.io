---
layout: post
title: 2022-07-28-Operator3-设计一个operator
date: 2022-07-28 2:00:00
category: kubernetes
tags: kubernetes operator
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
前置知识[Operator-1初识Operator](https://www.yuque.com/go/doc/81675278)，[Operator-2从pod开始简单operator](https://www.yuque.com/go/doc/82021143)。
先拿一个个人的工作环境来设计吧，应用有十多个微服务，恩各种类型job  deployment statefulset service ingress pv pvc  configmap这些资源准备模仿eck:[https://github.com/elastic/cloud-on-k8s](https://github.com/elastic/cloud-on-k8s)来设计！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657629249887-795d5982-7846-4049-8b87-af0aa735bcf1.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=859&id=ud776d62e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=773&originWidth=1278&originalType=binary&ratio=1&rotation=0&showTitle=false&size=185515&status=done&style=none&taskId=uf422c3d1-de18-4623-b28d-37ac85f96cf&title=&width=1420.0000376171545)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657629312599-59c08769-8efb-4164-b7cb-d731b6e395dc.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=594&id=u2bd09a65&margin=%5Bobject%20Object%5D&name=image.png&originHeight=535&originWidth=1055&originalType=binary&ratio=1&rotation=0&showTitle=false&size=127266&status=done&style=none&taskId=u1e82c2ec-6fb5-4640-9924-99de9022321&title=&width=1172.222253275507)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657629421335-19026ec0-78ba-423c-8a94-c477333eb6a4.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=322&id=uf833f9a7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=290&originWidth=1089&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57613&status=done&style=none&taskId=u140ccd45-2213-4950-a7ab-508fe37caec&title=&width=1210.0000320540544)
# 创建一个自己的operator
## goland 创建项目
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657629882999-91c3c1ac-78bd-4a35-9eca-dc3eee9aa09e.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=767&id=u2dce85d9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=690&originWidth=974&originalType=binary&ratio=1&rotation=0&showTitle=false&size=64425&status=done&style=none&taskId=u6f774abc-baae-4f54-8805-d3c7f5fd97e&title=&width=1082.2222508913212)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657629905881-c1a205fb-92f1-4702-989c-c2d4713186e4.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=619&id=u5f6ccbd7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=557&originWidth=1340&originalType=binary&ratio=1&rotation=0&showTitle=false&size=46540&status=done&style=none&taskId=uc923f8f4-f189-4b24-b068-0c5f31ba001&title=&width=1488.888928330976)
## 命名规则

关于应用命名一下吧：
就拿月份来作应用名称吧：
应用1：Jan 
应用2：feb  
应用3：mar 
应用4：apr 
应用5：may  
也没有想好具体的代表什么，下面了边作边看......
## kubebuilder init
```
[zhangpeng@zhangpeng develop-operator]$ kubebuilder init --plugins go/v3 --domain zhangpeng.com --owner "zhang peng"

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657630619586-d2934499-97f2-4e12-be75-1ac9a6780000.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=827&id=u04cb9c6c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=744&originWidth=1542&originalType=binary&ratio=1&rotation=0&showTitle=false&size=156671&status=done&style=none&taskId=u171a314d-4b99-4677-af81-621ea86e417&title=&width=1713.3333787211677)
## 开启支持多接口组
模仿目录结构：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658213815085-790c318d-ded6-4b11-b070-259ad651b776.png#clientId=uf0e18392-06d0-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=798&id=ue06c8a9a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=718&originWidth=1540&originalType=binary&ratio=1&rotation=0&showTitle=false&size=129823&status=done&style=none&taskId=u515643d2-0691-4dbf-80ba-0f7e0e3c30b&title=&width=1711.1111564400767)
设置multigroup=true，忘了这是在哪个地方搜到的了，应该是思否一篇文章

```
[zhangpeng@zhangpeng develop-operator]$ kubebuilder edit --multigroup=true
```
```
[zhangpeng@zhangpeng develop-operator]$ kubebuilder create api --group jan --version v1 --kind Jan

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657854449361-44575325-e3b7-4880-b001-9c95c62113ef.png#clientId=u713873c0-7b45-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=911&id=u1328f47b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=820&originWidth=1590&originalType=binary&ratio=1&rotation=0&showTitle=false&size=146506&status=done&style=none&taskId=u6b8b0f22-0238-4666-8d6b-691b3b77a81&title=&width=1766.6667134673519)
```
[zhangpeng@zhangpeng develop-operator]$ kubebuilder create api --group feb --version v1 --kind Feb

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657697384575-f5941ce0-8ece-43cd-bf4d-63980ab25720.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=982&id=ub149f785&margin=%5Bobject%20Object%5D&name=image.png&originHeight=884&originWidth=1308&originalType=binary&ratio=1&rotation=0&showTitle=false&size=146588&status=done&style=none&taskId=u6371e543-891f-487f-8f00-855f685382d&title=&width=1453.3333718335198)
```
[zhangpeng@zhangpeng develop-operator]$ kubebuilder create api --group mar --version v1 --kind Mar
[zhangpeng@zhangpeng develop-operator]$ kubebuilder create api --group apr --version v1 --kind Apr
[zhangpeng@zhangpeng develop-operator]$ kubebuilder create api --group may --version v1 --kind May
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657697542436-a79c1b13-4f89-4a06-a744-b3b13b1dbfde.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=816&id=u8a79db07&margin=%5Bobject%20Object%5D&name=image.png&originHeight=734&originWidth=1379&originalType=binary&ratio=1&rotation=0&showTitle=false&size=97662&status=done&style=none&taskId=ua208d194-d3c2-4113-94d7-8708fbae3b6&title=&width=1532.2222628122506)
# 从Jan开始
jan应用为一个deployment应用，参照[https://www.qikqiak.com/post/k8s-operator-101/](https://www.qikqiak.com/post/k8s-operator-101/),创建一个deployment并与Operator-2从pod开始简单operator中对比一下pod 与deployment的区别！
注意：以下代码都是抄写自阳明大佬，些许修改......，比如有个& 还有关于service的修改
## 定义jan_type
apis/jan/v1/jan_type.go
```
type JanSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Size      *int32                      `json:"size"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
}

// JanStatus defines the observed state of Jan
type JanStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	appsv1.DeploymentStatus `json:",inline"`
}
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657765938924-365cb89e-74c9-4ebd-adcf-5aaca358696f.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=846&id=u061b17fc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=761&originWidth=1475&originalType=binary&ratio=1&rotation=0&showTitle=false&size=187236&status=done&style=none&taskId=ud9feae39-7bf6-4b9f-a04b-640e6bab34d&title=&width=1638.888932304619)
## make install
make install 失败，继续拆解命令：

```
[zhangpeng@zhangpeng develop-operator]$ ./bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases

```

cat config/crd/bases/jan.zhangpeng.com_jans.yaml
```
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.8.0
  creationTimestamp: null
  name: jans.jan.zhangpeng.com
spec:
  group: jan.zhangpeng.com
  names:
    kind: Jan
    listKind: JanList
    plural: jans
    singular: jan
  scope: Namespaced
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        description: Jan is the Schema for the jans API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: JanSpec defines the desired state of Jan
            properties:
              envs:
                items:
                  description: EnvVar represents an environment variable present in
                    a Container.
                  properties:
                    name:
                      description: Name of the environment variable. Must be a C_IDENTIFIER.
                      type: string
                    value:
                      description: 'Variable references $(VAR_NAME) are expanded using
                        the previously defined environment variables in the container
                        and any service environment variables. If a variable cannot
                        be resolved, the reference in the input string will be unchanged.
                        Double $$ are reduced to a single $, which allows for escaping
                        the $(VAR_NAME) syntax: i.e. "$$(VAR_NAME)" will produce the
                        string literal "$(VAR_NAME)". Escaped references will never
                        be expanded, regardless of whether the variable exists or
                        not. Defaults to "".'
                      type: string
                    valueFrom:
                      description: Source for the environment variable's value. Cannot
                        be used if value is not empty.
                      properties:
                        configMapKeyRef:
                          description: Selects a key of a ConfigMap.
                          properties:
                            key:
                              description: The key to select.
                              type: string
                            name:
                              description: 'Name of the referent. More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names
                                TODO: Add other useful fields. apiVersion, kind, uid?'
                              type: string
                            optional:
                              description: Specify whether the ConfigMap or its key
                                must be defined
                              type: boolean
                          required:
                          - key
                          type: object
                        fieldRef:
                          description: 'Selects a field of the pod: supports metadata.name,
                            metadata.namespace, `metadata.labels[''<KEY>'']`, `metadata.annotations[''<KEY>'']`,
                            spec.nodeName, spec.serviceAccountName, status.hostIP,
                            status.podIP, status.podIPs.'
                          properties:
                            apiVersion:
                              description: Version of the schema the FieldPath is
                                written in terms of, defaults to "v1".
                              type: string
                            fieldPath:
                              description: Path of the field to select in the specified
                                API version.
                              type: string
                          required:
                          - fieldPath
                          type: object
                        resourceFieldRef:
                          description: 'Selects a resource of the container: only
                            resources limits and requests (limits.cpu, limits.memory,
                            limits.ephemeral-storage, requests.cpu, requests.memory
                            and requests.ephemeral-storage) are currently supported.'
                          properties:
                            containerName:
                              description: 'Container name: required for volumes,
                                optional for env vars'
                              type: string
                            divisor:
                              anyOf:
                              - type: integer
                              - type: string
                              description: Specifies the output format of the exposed
                                resources, defaults to "1"
                              pattern: ^(\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))))?$
                              x-kubernetes-int-or-string: true
                            resource:
                              description: 'Required: resource to select'
                              type: string
                          required:
                          - resource
                          type: object
                        secretKeyRef:
                          description: Selects a key of a secret in the pod's namespace
                          properties:
                            key:
                              description: The key of the secret to select from.  Must
                                be a valid secret key.
                              type: string
                            name:
                              description: 'Name of the referent. More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#names
                                TODO: Add other useful fields. apiVersion, kind, uid?'
                              type: string
                            optional:
                              description: Specify whether the Secret or its key must
                                be defined
                              type: boolean
                          required:
                          - key
                          type: object
                      type: object
                  required:
                  - name
                  type: object
                type: array
              image:
                type: string
              ports:
                items:
                  description: ServicePort contains information on service's port.
                  properties:
                    appProtocol:
                      description: The application protocol for this port. This field
                        follows standard Kubernetes label syntax. Un-prefixed names
                        are reserved for IANA standard service names (as per RFC-6335
                        and http://www.iana.org/assignments/service-names). Non-standard
                        protocols should use prefixed names such as mycompany.com/my-custom-protocol.
                      type: string
                    name:
                      description: The name of this port within the service. This
                        must be a DNS_LABEL. All ports within a ServiceSpec must have
                        unique names. When considering the endpoints for a Service,
                        this must match the 'name' field in the EndpointPort. Optional
                        if only one ServicePort is defined on this service.
                      type: string
                    nodePort:
                      description: 'The port on each node on which this service is
                        exposed when type is NodePort or LoadBalancer.  Usually assigned
                        by the system. If a value is specified, in-range, and not
                        in use it will be used, otherwise the operation will fail.  If
                        not specified, a port will be allocated if this Service requires
                        one.  If this field is specified when creating a Service which
                        does not need it, creation will fail. This field will be wiped
                        when updating a Service to no longer need it (e.g. changing
                        type from NodePort to ClusterIP). More info: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport'
                      format: int32
                      type: integer
                    port:
                      description: The port that will be exposed by this service.
                      format: int32
                      type: integer
                    protocol:
                      default: TCP
                      description: The IP protocol for this port. Supports "TCP",
                        "UDP", and "SCTP". Default is TCP.
                      type: string
                    targetPort:
                      anyOf:
                      - type: integer
                      - type: string
                      description: 'Number or name of the port to access on the pods
                        targeted by the service. Number must be in the range 1 to
                        65535. Name must be an IANA_SVC_NAME. If this is a string,
                        it will be looked up as a named port in the target Pod''s
                        container ports. If this is not specified, the value of the
                        ''port'' field is used (an identity map). This field is ignored
                        for services with clusterIP=None, and should be omitted or
                        set equal to the ''port'' field. More info: https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service'
                      x-kubernetes-int-or-string: true
                  required:
                  - port
                  type: object
                type: array
              resources:
                description: ResourceRequirements describes the compute resource requirements.
                properties:
                  limits:
                    additionalProperties:
                      anyOf:
                      - type: integer
                      - type: string
                      pattern: ^(\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))))?$
                      x-kubernetes-int-or-string: true
                    description: 'Limits describes the maximum amount of compute resources
                      allowed. More info: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/'
                    type: object
                  requests:
                    additionalProperties:
                      anyOf:
                      - type: integer
                      - type: string
                      pattern: ^(\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))(([KMGTPE]i)|[numkMGTPE]|([eE](\+|-)?(([0-9]+(\.[0-9]*)?)|(\.[0-9]+))))?$
                      x-kubernetes-int-or-string: true
                    description: 'Requests describes the minimum amount of compute
                      resources required. If Requests is omitted for a container,
                      it defaults to Limits if that is explicitly specified, otherwise
                      to an implementation-defined value. More info: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/'
                    type: object
                type: object
              size:
                format: int32
                maximum: 5
                minimum: 1
                type: integer
            type: object
          status:
            description: JanStatus defines the observed state of Jan
            properties:
              availableReplicas:
                description: Total number of available pods (ready for at least minReadySeconds)
                  targeted by this deployment.
                format: int32
                type: integer
              collisionCount:
                description: Count of hash collisions for the Deployment. The Deployment
                  controller uses this field as a collision avoidance mechanism when
                  it needs to create the name for the newest ReplicaSet.
                format: int32
                type: integer
              conditions:
                description: Represents the latest available observations of a deployment's
                  current state.
                items:
                  description: DeploymentCondition describes the state of a deployment
                    at a certain point.
                  properties:
                    lastTransitionTime:
                      description: Last time the condition transitioned from one status
                        to another.
                      format: date-time
                      type: string
                    lastUpdateTime:
                      description: The last time this condition was updated.
                      format: date-time
                      type: string
                    message:
                      description: A human readable message indicating details about
                        the transition.
                      type: string
                    reason:
                      description: The reason for the condition's last transition.
                      type: string
                    status:
                      description: Status of the condition, one of True, False, Unknown.
                      type: string
                    type:
                      description: Type of deployment condition.
                      type: string
                  required:
                  - status
                  - type
                  type: object
                type: array
              observedGeneration:
                description: The generation observed by the deployment controller.
                format: int64
                type: integer
              readyReplicas:
                description: readyReplicas is the number of pods targeted by this
                  Deployment with a Ready Condition.
                format: int32
                type: integer
              replicas:
                description: Total number of non-terminated pods targeted by this
                  deployment (their labels match the selector).
                format: int32
                type: integer
              unavailableReplicas:
                description: Total number of unavailable pods targeted by this deployment.
                  This is the total number of pods that are still required for the
                  deployment to have 100% available capacity. They may either be pods
                  that are running but not yet available or pods that still have not
                  been created.
                format: int32
                type: integer
              updatedReplicas:
                description: Total number of non-terminated pods targeted by this
                  deployment that have the desired template spec.
                format: int32
                type: integer
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []

```
继续发布到集群：
```
[zhangpeng@zhangpeng develop-operator]$ kustomize build config/crd | kubectl apply -f -

```
不知道有没有单独发布的办法，一整都一起发布了......
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1657767647616-dc54467e-506b-4e6d-a09d-3729ba5da01c.png#clientId=uaa6cdef0-c963-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1018&id=udfa27d48&margin=%5Bobject%20Object%5D&name=image.png&originHeight=916&originWidth=1675&originalType=binary&ratio=1&rotation=0&showTitle=false&size=254080&status=done&style=none&taskId=u8860ef50-a47c-4852-8a53-0f84e354b80&title=&width=1861.1111604137197)
```
[zhangpeng@zhangpeng develop-operator]$ kubectl describe crd jans.jan.zhangpeng.com 

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658110159691-1a3892c5-20ae-4260-93ba-b90d569b4c56.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=656&id=u004c37d1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=590&originWidth=1287&originalType=binary&ratio=1&rotation=0&showTitle=false&size=101846&status=done&style=none&taskId=udf9557d7-7e94-4b45-8b5a-0fb4c323df1&title=&width=1430.0000378820641)
describe crd的内容与config/crd/bases/jan.zhangpeng.com_jans.yaml 内容是一样的，强调一下......
## 创建deployment  pod service的方法
对应文件都偷懒了，直接放在controllers/jan目录下了：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658116494060-607e6cd5-d3d0-4227-bc89-91b680e04d2f.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=830&id=u7cd7dfb5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=747&originWidth=1264&originalType=binary&ratio=1&rotation=0&showTitle=false&size=164064&status=done&style=none&taskId=ueb2c8132-27ce-4e0c-8016-80863b5e554&title=&width=1404.4444816495175)
cat jan_helper.go
```
package jan

import (
	janv1 "develop-operator/apis/jan/v1"
	appv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

func NewJan(app *janv1.Jan) *appv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			Kind:       "apps/v1",
			APIVersion: "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: appv1.DeploymentSpec{
			Replicas: app.Spec.Size,
			Selector: selector,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec:       corev1.PodSpec{Containers: newContainers(app)},
			},
			MinReadySeconds: 0,
		},
	}

}
func newContainers(app *janv1.Jan) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name:            app.Name,
			Image:           app.Spec.Image,
			Resources:       app.Spec.Resources,
			Ports:           containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env:             app.Spec.Envs,
		},
	}
}
func NewService(app *janv1.Jan) *corev1.Service {
	return &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type:  corev1.ServiceTypeNodePort,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}

```
## jan_controller.go   Reconcile
基本阳明大佬的博客抄来的,Reconcile调谐函数：
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
	"k8s.io/apimachinery/pkg/api/errors"
	"reflect"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	janv1 "develop-operator/apis/jan/v1"
)

// JanReconciler reconciles a Jan object
type JanReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=jan.zhangpeng.com,resources=jans,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=jan.zhangpeng.com,resources=jans/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=jan.zhangpeng.com,resources=jans/finalizers,verbs=update

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
		// 2. 创建 Service
		service := NewService(instance)
		if err := r.Client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 关联 Annotations
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

	oldspec := janv1.JanSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), &oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
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

	return reconcile.Result{}, nil

}

// SetupWithManager sets up the controller with the Manager.
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Complete(r)
}

```
## 强调一下：
json.Unmarshal
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658116643398-54ba61fc-8e54-48ae-8789-c569a4a0cdb2.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=468&id=u7e806497&margin=%5Bobject%20Object%5D&name=image.png&originHeight=421&originWidth=1552&originalType=binary&ratio=1&rotation=0&showTitle=false&size=98273&status=done&style=none&taskId=ud3473f04-dc10-43c4-9dbc-ce829a3825e&title=&width=1724.444490126623)
[https://www.qikqiak.com/post/k8s-operator-101/#%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84](https://www.qikqiak.com/post/k8s-operator-101/#%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84)是这样写的，but maker run就报错了，看了一下别人有人写了&就加了一下
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125100256-37aea3a7-064a-4dfc-b1cd-06b9831c1679.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=153&id=uea5c8619&margin=%5Bobject%20Object%5D&name=image.png&originHeight=138&originWidth=1131&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21181&status=done&style=none&taskId=u10847ba3-3d55-40f5-8594-3f943b5fcf1&title=&width=1256.6666999569654)
## make run and test
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125458511-7ec04abd-97d7-41b7-8128-226c99722865.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=823&id=u30273689&margin=%5Bobject%20Object%5D&name=image.png&originHeight=741&originWidth=1593&originalType=binary&ratio=1&rotation=0&showTitle=false&size=193585&status=done&style=none&taskId=u242f5c33-ba71-427f-9d30-e3c13662690&title=&width=1770.0000468889884)
测试yaml就用config/samples/jan_v1_jan.yaml去测试了：
```
apiVersion: jan.zhangpeng.com/v1
kind: Jan
metadata:
  name: jan-sample
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002
```
```
[zhangpeng@zhangpeng develop-operator]$ kubectl apply -f config/samples/jan_v1_jan.yaml 
jan.jan.zhangpeng.com/jan-sample created

```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125444412-d37929d4-6baf-4e30-bbd4-d5a4348e3a6a.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=972&id=ufe13c227&margin=%5Bobject%20Object%5D&name=image.png&originHeight=875&originWidth=1203&originalType=binary&ratio=1&rotation=0&showTitle=false&size=183260&status=done&style=none&taskId=u52751795-8646-4922-aab0-34f3f760f7c&title=&width=1336.6667020762418)
修改副本数为3：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125497436-f98ed247-888c-4420-83d7-a1ade41ca005.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=970&id=u90c8e31d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=873&originWidth=1313&originalType=binary&ratio=1&rotation=0&showTitle=false&size=182891&status=done&style=none&taskId=uf0c9c6c4-03dc-4556-a8c8-59d2433eec2&title=&width=1458.8889275362471)
# 继续改造
## 修改Service Type
en 对比前一节的pod  operator我想输出更多的内容，get Jan也想输出数量：继续改造(还有Jan服务可以输入Type我不喜欢nodePort的方式)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125876484-f0970edb-bbbd-4400-a8dc-0193d18a3661.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=749&id=u47e966f5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=674&originWidth=1545&originalType=binary&ratio=1&rotation=0&showTitle=false&size=177028&status=done&style=none&taskId=u144fbc5f-20c4-4c95-a72d-88c9aee5c1d&title=&width=1716.6667121428043)
```
[zhangpeng@zhangpeng develop-operator]$ make install
[zhangpeng@zhangpeng develop-operator]$ ./bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
[zhangpeng@zhangpeng develop-operator]$  kustomize build config/crd | kubectl apply -f -

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658125949436-8309ece2-d576-4cd1-b31c-1e39bddf7311.png#clientId=u956409ac-813e-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=327&id=uaaa16e58&margin=%5Bobject%20Object%5D&name=image.png&originHeight=294&originWidth=1557&originalType=binary&ratio=1&rotation=0&showTitle=false&size=84798&status=done&style=none&taskId=ue663eb22-1b75-4de1-ab18-849e7a35963&title=&width=1730.0000458293503)
kubecelt delete -f config/samples/jan_v1_jan.yaml
重新编辑文件如下：
```

apiVersion: jan.zhangpeng.com/v1
kind: Jan
metadata:
  name: jan-sample
spec:
  size: 3
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
//     nodePort: 30002
    type: ClusterIP
```
controllers/jan/jan_helper.go  NewService修改Type如下：**Type:  app.Spec.Type,**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658980341264-4f067c29-3248-44b1-853e-74361cbd4d11.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=769&id=ub64c9a78&margin=%5Bobject%20Object%5D&name=image.png&originHeight=692&originWidth=1503&originalType=binary&ratio=1&rotation=0&showTitle=false&size=153429&status=done&style=none&taskId=ua2e9fb46-f29f-41d3-8e8f-4bdb4882d9a&title=&width=1670.000044239893)
make run   and kubectl apply
```
[zhangpeng@zhangpeng develop-operator]$ kubectl delete -f config/samples/jan_v1_jan.yaml 
jan.jan.zhangpeng.com "jan-sample" deleted
[zhangpeng@zhangpeng develop-operator]$ kubectl apply -f config/samples/jan_v1_jan.yaml 
jan.jan.zhangpeng.com/jan-sample created
[zhangpeng@zhangpeng develop-operator]$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
jan-sample   ClusterIP   10.99.27.123   <none>        80/TCP    1s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   19d
[zhangpeng@zhangpeng develop-operator]$ kubectl get jan
NAME         AGE
jan-sample   5m21s
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658307194375-b5cd5bea-1e63-44f9-a416-17e5aeb87aa0.png#clientId=u796076b1-9323-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=302&id=u04f44f9c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=272&originWidth=1064&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59512&status=done&style=none&taskId=u26d7ee94-39ad-4d1b-9f7a-551036f0e40&title=&width=1182.2222535404167)
## 继续模仿  
eck 有更多的输出阿   咱们的输出现在只有AGE......想输出更多
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658233724631-d4fb83eb-06ae-4dd5-996f-384bed87aa44.png#clientId=u340c8671-026c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=149&id=u9bd5c753&margin=%5Bobject%20Object%5D&name=image.png&originHeight=134&originWidth=1029&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34471&status=done&style=none&taskId=u87c6a7f3-07ec-4590-955e-33cb7a71152&title=&width=1143.333363621324)
注意：comon我还是没有用到，创建了就创建了吧，后面看看还是否用的到！
人家eck有个公用的comon?咱也搞一个
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658233747687-d513bfc0-1d4f-4488-85e6-be7567e8625c.png#clientId=u340c8671-026c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1009&id=u318029fe&margin=%5Bobject%20Object%5D&name=image.png&originHeight=908&originWidth=1413&originalType=binary&ratio=1&rotation=0&showTitle=false&size=229862&status=done&style=none&taskId=u73db76cb-dbb4-4cb8-bc55-cea64aaa178&title=&width=1570.0000415907978)
```
kubebuilder create api --group common --version v1 --kind Common
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658233862765-d56de30c-a338-4695-999f-e4ab37eb673c.png#clientId=u340c8671-026c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=486&id=ub5f84024&margin=%5Bobject%20Object%5D&name=image.png&originHeight=437&originWidth=1516&originalType=binary&ratio=1&rotation=0&showTitle=false&size=61824&status=done&style=none&taskId=ua1eec34f-f9df-4a57-8007-85bcb6dfeab&title=&width=1684.4444890669847)
继续狗一下
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658234006529-5fd542d3-d6f4-47e2-b557-de8309f573bb.png#clientId=u340c8671-026c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=782&id=uc9936344&margin=%5Bobject%20Object%5D&name=image.png&originHeight=704&originWidth=1496&originalType=binary&ratio=1&rotation=0&showTitle=false&size=159544&status=done&style=none&taskId=ucb427f30-9e44-419d-a574-8b32637d8a7&title=&width=1662.2222662560746)
这里懒得写了 网上搜到一个博客：[https://qingwave.github.io/how-to-write-a-k8s-operator](https://qingwave.github.io/how-to-write-a-k8s-operator)就按照他写的改一下了：
apis/jan/v1/jan_type.go
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

package v1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// JanSpec defines the desired state of Jan
type JanSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	//+kubebuilder:default:=1
	//+kubebuilder:validation:Minimum:=1
	Replicas  *int32                      `json:"replicas,omitempty" protobuf:"varint,1,opt,name=replicas"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
	Type      corev1.ServiceType          `json:"type,omitempty"`
}

const (
	Running  = "Running"
	Pending  = "Pending"
	NotReady = "NotReady"
	Failed   = "Failed"
)

// JanStatus defines the observed state of Jan
type JanStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	// Phase is the phase of guestbook
	Phase string `json:"phase,omitempty"`
	// replicas is the number of Pods created by the StatefulSet controller.
	Replicas int32 `json:"replicas"`

	// readyReplicas is the number of Pods created by the StatefulSet controller that have a Ready Condition.
	ReadyReplicas int32 `json:"readyReplicas"`

	// LabelSelector is label selectors for query over pods that should match the replica count used by HPA.
	LabelSelector string `json:"labelSelector,omitempty"`
}

//+kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas,selectorpath=.status.labelSelector
//+kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase",description="The phase of game."
//+kubebuilder:printcolumn:name="DESIRED",type="integer",JSONPath=".spec.replicas",description="The desired number of pods."
//+kubebuilder:printcolumn:name="CURRENT",type="integer",JSONPath=".status.replicas",description="The number of currently all pods."
//+kubebuilder:printcolumn:name="READY",type="integer",JSONPath=".status.readyReplicas",description="The number of pods ready."
//+kubebuilder:printcolumn:name="AGE",type="date",JSONPath=".metadata.creationTimestamp",description="CreationTimestamp is a timestamp representing the server time when this object was created. It is not guaranteed to be set in happens-before order across separate operations. Clients may not set this value. It is represented in RFC3339 form and is in UTC."
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// Jan is the Schema for the jans API
type Jan struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   JanSpec   `json:"spec,omitempty"`
	Status JanStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// JanList contains a list of Jan
type JanList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Jan `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Jan{}, &JanList{})
}

```
make install 
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658818160976-7f78c143-0a9a-4d6e-9f1f-c99408ba6cf0.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=257&id=ud5c5d2dc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=231&originWidth=1565&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57976&status=done&style=none&taskId=u3a4c0cc6-cf86-4d16-9264-719ba3b8169&title=&width=1738.8889349537144)
controllers/jan/jan_helper.go
其实我就修改了一下type，我可不想一直写nodeport......一般都司clasterip
```
package jan

import (
	janv1 "develop-operator/apis/jan/v1"
	appv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

func NewJan(app *janv1.Jan) *appv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			Kind:       "apps/v1",
			APIVersion: "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: appv1.DeploymentSpec{
			Replicas: app.Spec.Replicas,
			Selector: selector,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec:       corev1.PodSpec{Containers: newContainers(app)},
			},
			MinReadySeconds: 0,
		},
		Status: appv1.DeploymentStatus{},
	}

}

func newContainers(app *janv1.Jan) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name:            app.Name,
			Image:           app.Spec.Image,
			Resources:       app.Spec.Resources,
			Ports:           containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env:             app.Spec.Envs,
		},
	}
}

func NewService(app *janv1.Jan) *corev1.Service {
	return &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type:  app.Spec.Type,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}

```
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

		// 2. 创建 Service
		service := NewService(instance)
		if err := r.Client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 关联 Annotations
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
	oldspec := janv1.JanSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), &oldspec); err != nil {
		return reconcile.Result{}, err
	}
	if !reflect.DeepEqual(instance.Spec, oldspec) {
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
		err := r.Client.Status().Update(ctx, instance)
		return reconcile.Result{}, err

	}
	return reconcile.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *JanReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&janv1.Jan{}).
		Complete(r)
}

```
偷懒抄来的：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658819510684-76434716-8c04-4639-9bae-bab58d4f37a1.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=426&id=ufeb14fe8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=383&originWidth=1062&originalType=binary&ratio=1&rotation=0&showTitle=false&size=56861&status=done&style=none&taskId=u31fc169f-53a0-4e7f-985b-623eb7d3b0a&title=&width=1180.0000312593256)
make run :


清空原来的Jan应用
```
[zhangpeng@zhangpeng develop-operator]$ kubectl delete jan jan-sample
jan.jan.zhangpeng.com "jan-sample" deleted

```
```
apiVersion: jan.zhangpeng.com/v1
kind: Jan
metadata:
  name: jan-sample
spec:
  replicas: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658819730689-169c80b9-f3aa-4de6-982d-7424918b24c4.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=197&id=u2d1c0139&margin=%5Bobject%20Object%5D&name=image.png&originHeight=177&originWidth=1411&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42522&status=done&style=none&taskId=u61bc5e76-60a5-42b3-b13e-405b79f04a0&title=&width=1567.7778193097067)
```
[zhangpeng@zhangpeng develop-operator]$ kubectl apply -f config/samples/jan_v1_jan.yaml 
jan.jan.zhangpeng.com/jan-sample created
[zhangpeng@zhangpeng develop-operator]$ kubectl get jan
NAME         PHASE     DESIRED   CURRENT   READY   AGE
jan-sample   Running   2         2         2       1s
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658819705451-03623e31-f4ea-4a59-a062-43c781138aa8.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=836&id=uaebafc2f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=752&originWidth=1722&originalType=binary&ratio=1&rotation=0&showTitle=false&size=255203&status=done&style=none&taskId=u6293ee7c-e583-45d1-9f56-affc9221a18&title=&width=1913.3333840193586)
## 问题又来了：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658819834156-d623bd97-13a8-474d-b16a-399ddc132f78.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1014&id=u4b8b13ba&margin=%5Bobject%20Object%5D&name=image.png&originHeight=913&originWidth=1462&originalType=binary&ratio=1&rotation=0&showTitle=false&size=184111&status=done&style=none&taskId=u5c811cd7-c53e-4f02-9177-0b23c1652f4&title=&width=1624.4444874775274)
这样还是不能更新状态status阿？没有什么实际意义。所以这个哥们写的也是一个纯看的demo....还是找个成熟的应用去看吧！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658890000052-04e56988-4232-4b15-88e9-5f549fa01477.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=894&id=u91862e88&margin=%5Bobject%20Object%5D&name=image.png&originHeight=805&originWidth=996&originalType=binary&ratio=1&rotation=0&showTitle=false&size=114115&status=done&style=none&taskId=uc5f1869e-ed96-4b2d-b0e6-7afbf651c54&title=&width=1106.6666959833224)
## 偷懒秘籍：
更新资源这里我是不是可以重新生成一下Annotations？那旧的数据不就变成新的了？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658891306995-47cd33c5-101b-42b7-8aaf-d7a7e8d02021.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=878&id=u7d9fd2ea&margin=%5Bobject%20Object%5D&name=image.png&originHeight=790&originWidth=1245&originalType=binary&ratio=1&rotation=0&showTitle=false&size=205296&status=done&style=none&taskId=uaf568a17-f2fa-4aea-a930-c4f10d54523&title=&width=1383.3333699791528)
尝试一下：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658892298344-34f01b54-448d-42c9-86b2-8f1706670311.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=811&id=u128d97b1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=730&originWidth=1495&originalType=binary&ratio=1&rotation=0&showTitle=false&size=147031&status=done&style=none&taskId=ue47bd30b-2dbe-4d60-af96-56a5445301f&title=&width=1661.111155115529)
自己骗自己算是成功了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658892331587-c2caecd1-6b0f-40d3-a36f-ae48132f8968.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=840&id=u6daf570d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=756&originWidth=1602&originalType=binary&ratio=1&rotation=0&showTitle=false&size=111667&status=done&style=none&taskId=u5233ffc7-86d8-4481-812f-e841815569b&title=&width=1780.000047153898)
但是还是有问题：
修改image
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658892515039-2c608e0c-e213-4939-9fb3-bacab6db4a46.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=602&id=ud6ec5939&margin=%5Bobject%20Object%5D&name=image.png&originHeight=542&originWidth=1205&originalType=binary&ratio=1&rotation=0&showTitle=false&size=89057&status=done&style=none&taskId=u707e3225-fed8-4c16-bb18-ae05a681256&title=&width=1338.8889243573328)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658892550126-0830b461-a8f6-4b82-8054-6a055b269748.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=307&id=u049fdca9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=276&originWidth=819&originalType=binary&ratio=1&rotation=0&showTitle=false&size=58286&status=done&style=none&taskId=u0481f780-c09b-4baa-8e22-01cc282d6dc&title=&width=910.0000241067681)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658892565898-fc218b3b-fec1-46f1-884d-3b5a78ea64b5.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=786&id=u8c422b96&margin=%5Bobject%20Object%5D&name=image.png&originHeight=707&originWidth=1401&originalType=binary&ratio=1&rotation=0&showTitle=false&size=97505&status=done&style=none&taskId=u1122f570-7bf2-4210-a94c-557faef5310&title=&width=1556.6667079042516)
pod更新中这样是否能接受呢？
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658893120616-ef7bb88b-c6d3-40f4-bcb2-1733c7022fd3.png#clientId=ud715770b-f39c-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=800&id=u288a2836&margin=%5Bobject%20Object%5D&name=image.png&originHeight=720&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&size=136204&status=done&style=none&taskId=u88bc4296-ed18-4b98-948b-c6607e7690c&title=&width=1268.888922502966)
看了一眼deployment也是这样，我接受了.......
# 最终代码
上面感觉还是缺少点东西，什么呢？ingress我是否也可以封过来？
## 增加ingress相关字段
apis/jan/v1/jan_type.go
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658975177262-872ee8b6-297b-4537-90ef-02fe03ae6f37.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=644&id=u350aa309&margin=%5Bobject%20Object%5D&name=image.png&originHeight=580&originWidth=1186&originalType=binary&ratio=1&rotation=0&showTitle=false&size=115491&status=done&style=none&taskId=ud51ec2b7-a4da-410b-a354-c027ba6f7f2&title=&width=1317.7778126869682)
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

package v1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// JanSpec defines the desired state of Jan
type JanSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	//+kubebuilder:default:=1
	//+kubebuilder:validation:Minimum:=1
	Replicas  *int32                      `json:"replicas,omitempty" protobuf:"varint,1,opt,name=replicas"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
	Type      corev1.ServiceType          `json:"type,omitempty"`
	Host      string                      `json:"host,omitempty"`
}

const (
	Running  = "Running"
	Pending  = "Pending"
	NotReady = "NotReady"
	Failed   = "Failed"
)

// JanStatus defines the observed state of Jan
type JanStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	// Phase is the phase of guestbook
	Phase string `json:"phase,omitempty"`
	// replicas is the number of Pods created by the StatefulSet controller.
	Replicas int32 `json:"replicas"`

	// readyReplicas is the number of Pods created by the StatefulSet controller that have a Ready Condition.
	ReadyReplicas int32 `json:"readyReplicas"`

	// LabelSelector is label selectors for query over pods that should match the replica count used by HPA.
	LabelSelector string `json:"labelSelector,omitempty"`
}

//+kubebuilder:printcolumn:name="Host",type="string",JSONPath=".spec.host",description="The host address."
//+kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas,selectorpath=.status.labelSelector
//+kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase",description="The phase of game."
//+kubebuilder:printcolumn:name="DESIRED",type="integer",JSONPath=".spec.replicas",description="The desired number of pods."
//+kubebuilder:printcolumn:name="CURRENT",type="integer",JSONPath=".status.replicas",description="The number of currently all pods."
//+kubebuilder:printcolumn:name="READY",type="integer",JSONPath=".status.readyReplicas",description="The number of pods ready."
//+kubebuilder:printcolumn:name="AGE",type="date",JSONPath=".metadata.creationTimestamp",description="CreationTimestamp is a timestamp representing the server time when this object was created. It is not guaranteed to be set in happens-before order across separate operations. Clients may not set this value. It is represented in RFC3339 form and is in UTC."
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// Jan is the Schema for the jans API
type Jan struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   JanSpec   `json:"spec,omitempty"`
	Status JanStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// JanList contains a list of Jan
type JanList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Jan `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Jan{}, &JanList{})
}

```
## 创建NewIngress方法
jan_helper.go 中，先写死这个了port了
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658975287612-f1764b5e-0274-4357-9beb-c00b8677d324.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=937&id=uab3f450c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=843&originWidth=1219&originalType=binary&ratio=1&rotation=0&showTitle=false&size=110127&status=done&style=none&taskId=ub17ab7fa-901f-4e1a-b95a-0ab8afa0a9c&title=&width=1354.4444803249698)
```
package jan

import (
	janv1 "develop-operator/apis/jan/v1"
	appv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/api/networking/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

func NewJan(app *janv1.Jan) *appv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			Kind:       "apps/v1",
			APIVersion: "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: appv1.DeploymentSpec{
			Replicas: app.Spec.Replicas,
			Selector: selector,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{Labels: labels},
				Spec:       corev1.PodSpec{Containers: newContainers(app)},
			},
			MinReadySeconds: 0,
		},
		Status: appv1.DeploymentStatus{},
	}

}

func newContainers(app *janv1.Jan) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name:            app.Name,
			Image:           app.Spec.Image,
			Resources:       app.Spec.Resources,
			Ports:           containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env:             app.Spec.Envs,
		},
	}
}

func NewService(app *janv1.Jan) *corev1.Service {
	return &corev1.Service{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group:   janv1.GroupVersion.Group,
					Version: janv1.GroupVersion.Version,
					Kind:    "Jan",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type:  app.Spec.Type,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}

const (
	port = 80
)

func NewIngress(app *janv1.Jan) *v1.Ingress {
	pathType := v1.PathTypePrefix
	return &v1.Ingress{
		TypeMeta: metav1.TypeMeta{
			Kind:       "Ingress",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,
		},
		Spec: v1.IngressSpec{
			IngressClassName: nil,
			Rules: []v1.IngressRule{
				{
					Host: app.Spec.Host,
					IngressRuleValue: v1.IngressRuleValue{
						HTTP: &v1.HTTPIngressRuleValue{
							Paths: []v1.HTTPIngressPath{{
								Path:     "/",
								PathType: &pathType,
								Backend: v1.IngressBackend{
									Service: &v1.IngressServiceBackend{
										Name: app.Name,
										Port: v1.ServiceBackendPort{
											Number: int32(port),
										},
									},
									Resource: nil,
								},
							},
							}}},
				},
			},
		},
	}
}

```
jan_controller.go中增加ingress相关

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658981557793-30d1729f-f3d6-4829-9677-ddcaca805552.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=411&id=u203c5a22&margin=%5Bobject%20Object%5D&name=image.png&originHeight=370&originWidth=1349&originalType=binary&ratio=1&rotation=0&showTitle=false&size=86951&status=done&style=none&taskId=u51a1267f-b436-4389-8859-b6f87c2197d&title=&width=1498.8889285958853)
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

		// 2. 创建 Service
		service := NewService(instance)
		if err := r.Client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 创建 Ingress
		ingress := NewIngress(instance)
		if err := r.Client.Create(context.TODO(), ingress); err != nil {
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
		Complete(r)
}

```
## 实验一下：
**make install  make run**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658975542721-7bb02b03-c926-4d70-aa12-340bcebaa5bf.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=534&id=uc99c2b9c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=481&originWidth=1615&originalType=binary&ratio=1&rotation=0&showTitle=false&size=133808&status=done&style=none&taskId=ue034e94e-df51-40c4-ac47-6eed63d5791&title=&width=1794.4444919809896)
删除清空jan应用
```
[zhangpeng@zhangpeng develop-operator]$ kubectl delete -f config/samples/jan_v1_jan.yaml 
jan.jan.zhangpeng.com "jan-sample" deleted

```
修改config/samples/jan_v1_jan.yaml 如下：
```
apiVersion: jan.zhangpeng.com/v1
kind: Jan
metadata:
  name: jan-sample
spec:
  replicas: 3
  image: nginx:1.17.6
  host: www.zhangpeng.com
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658975693508-d34d00b8-f823-4666-b5b2-814163a6bedf.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1033&id=uc2ca389f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=930&originWidth=1388&originalType=binary&ratio=1&rotation=0&showTitle=false&size=183829&status=done&style=none&taskId=u5b517eeb-862b-4e55-b6fa-8a8322ec954&title=&width=1542.22226307716)
接着修改config/samples/jan_v1_jan.yaml 副本数与host
```
apiVersion: jan.zhangpeng.com/v1
kind: Jan
metadata:
  name: jan-sample
spec:
  replicas: 2
  image: nginx:1.17.6
  host: www1.zhangpeng.com
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/2505271/1658975766077-8c31e081-6148-4961-8133-88b02bcabc3b.png#clientId=u9ed93ae2-c4a5-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1029&id=ubf3b448d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=926&originWidth=1512&originalType=binary&ratio=1&rotation=0&showTitle=false&size=200902&status=done&style=none&taskId=ueb4f22bb-2ac8-4cad-ac08-4adcaa75132&title=&width=1680.0000445048026)
基本实现了第一步的需求了！
# 总结一下：

1. operator要解决的是什么 自己还是没有搞明确，也没有想好怎么去设计一个operator。只是简单的实现了一些基本的功能，还没有体会到更多的便利性。
1. 本来想照着eck写，但是对我这种初学者还是有点难，一步一步 去完善写吧......
