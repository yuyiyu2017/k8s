## 1、概念
**准入控制器（Admission Controller）**位于 API Server 中，在对象被持久化之前，准入控制器拦截对 API Server 的请求，一般用来做身份验证和授权。其中包含两个特殊的控制器：MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。
>**变更（Mutating）准入控制：**修改请求的对象
>
>**验证（Validating）准入控制：**验证请求的对象

## 2、配置解释

```
# 查看所有可用插件
kube-apiserver -h | grep enable-admission-plugins
```
* **AlwaysAdmit：**允许所有请求通过
* **AlwaysDeny：**禁止所有请求通过
* **AlwaysPullImages：**
```
在启动容器之前总是去下载镜像，相当于每当容器启动前做一次用于是否有权使用该容器镜像的检查
```
* **DefaultStorageClass：**
```
这个插件观察不指定 storage class 字段的 PersistentVolumeClaim 对象的创建，并自动向它们添加默认的 storage class，因此不指定 storage class 字段的用户根本无需关心它们，它们将得到默认的 storage class

当没有配置默认 storage class 时，这个插件不会执行任何操作
当一个以上的 storage class 被标记为默认时，它拒绝 PersistentVolumeClaim 创建并返回一个错误，管理员必须重新检查 StorageClass 对象，并且只标记一个作为默认值

这个插件忽略了任何 PersistentVolumeClaim 更新，它只对创建起作用
```
* **DefaultTolerationSeconds：**
```
将Pod的容忍时间notready:NoExecute和unreachable:NoExecute 默认设置为5分钟
```
* **DenyEscalatingExec：**
```
拒绝在拥有衍生特权而具备访问宿主机能力的 pod 中执行 exec 和 attach 命令
包括在特权模式运行的 pod ，可以访问主机 IPC 名称空间的 pod ，和访问主机 PID 命名空间的 pod

如果集群支持使用以衍生特权运行的容器，并且希望限制最终用户在这些容器中执行 exec 命令的能力，建议启用这个插件
```
* **EventRateLimit：**
```
限制 API Server 被事件请求的频率，防止高负荷
```
* **ExtendedResourceToleration：**
```
此插件有助于创建具有扩展资源的专用节点
```
* **ImagePolicyWebhook：**
```
此准入控制器允许后端判断镜像拉取策略，例如配置镜像仓库的密钥
```
* **Initializers：**
```
Pod初始化的准入控制器，详情请参考动态准入控制
```
* **LimitPodHardAntiAffinityTopology：**
```
拒绝任何在 requiredDuringSchedulingRequiredDuringExecution 的 AntiAffinity 字段中定义除了kubernetes.io/hostname 之外的拓扑关键字的 pod
```
* **LimitRanger：**
```
这个插件将观察传入的请求，并确保它不会违反 Namespace 中 LimitRange 对象枚举的任何约束

如果在 Kubernetes 部署中使用了 LimitRange 对象，则必须使用此插件来执行这些约束

LimitRanger 插件还可以用于将默认资源请求应用到没有指定任何内容的 Pod
当前，默认的 LimitRanger 对 default 名称空间中的所有 pod 应用了0.1 CPU 的需求
```
* **MutatingAdmissionWebhook：**
```
该准入控制器调用与请求匹配的任何变更 webhook
匹配的 webhook是串行调用的；如果需要，每个人都可以修改对象
```
* **NamespaceAutoProvision：**
```
检查命名空间资源上的所有传入请求，并检查引用的命名空间是否存在
如果不存在就创建一个命名空间
```
* **NamespaceExists：**
```
此许可控制器检查除 Namespace 其自身之外的命名空间资源上的所有请求
如果请求引用的命名空间不存在，则拒绝该请求
```
* **NamespaceLifecycle：**
```
当一个请求是在一个不存在的namespace下创建资源对象时，该请求会被拒绝
当删除一个namespace时，将会删除该namespace下的所有资源对象
此准入控制器还防止缺失三个系统保留的命名空间default、kube-system、kube-public
```
* **NodeRestriction：**
```
这个插件限制了 kubelet 可以修改的 Node 和 Pod 对象
kubelet 必须在 system：nodes 组中使用凭证，并使用 system：node：<nodeName> 形式的用户名
这样的 kubelet 只允许修改自己的 Node API 对象，只能修改绑定到节点本身的 Pod 对象
```
* **OwnerReferencesPermissionEnforcement：**
```
保护对metadata.ownerReferences对象的访问，以便只有对该对象具有“删除”权限的用户才能对其进行更改
```
* **PodNodeSelector：**
```
此准入控制器通过读取命名空间注释和全局配置，限制可在命名空间内使用的节点选择器
```
* **PodPreset：**
```
此准入控制器注入一个pod，其中包含匹配的PodPreset中指定的字段，详细信息见Pod Preset
```
* **PodSecurityPolicy：**
```
此插件负责在创建和修改 pod 时，根据请求的安全上下文和可用的 pod 安全策略确定是否应该通过 pod
```
* **PodTolerationRestriction：**
```
此准入控制器首先验证容器的容忍度与其命名空间的容忍度之间是否存在冲突，并在存在冲突时拒绝该容器请求
```
* **Priority：**
```
此控制器使用priorityClassName字段并填充优先级的整数值
如果未找到优先级，则拒绝Pod
```
* **ResourceQuota：**
```
用于namespace上的配额管理，它会观察进入的请求，确保在namespace上的配额不超标
```
* **SecurityContextDeny：**
```
将Pod定义中定义了的SecurityContext选项全部失效，拒绝任何试图设置某些升级的SecurityContext字段的pod

SecurityContext包含在容器中定义了操作系统级别的安全选型如fsGroup，selinux等选项
```

* **ServiceAccount：**
```
这个插件实现了serviceAccounts等自动化
如果使用ServiceAccount对象，推荐使用这个插件
```
* **PersistentVolumeClaimResize：**
```
该StorageObjectInUseProtection插件将kubernetes.io/pvc-protection或kubernetes.io/pv-protection终结器添加到新创建的持久卷声明（PVC）或持久卷（PV）
在用户删除PVC或PV的情况下，PVC或PV不会被移除，直到PVC或PV保护控制器从PVC或PV中移除终结器
```
* **ValidatingAdmissionWebhook：**
```
该准入控制器调用与请求匹配的任何验证webhook
匹配的webhooks是并行调用的；如果其中任何一个拒绝请求，则请求失败
```

## 3、常用默认配置
>1.10版本及以上
```bash
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

## 4、SecurityContextDeny报错
```
SecurityContext.RunAsUser is forbidden
```

## 5、ImagePolicyWebhook配置
### a、配置文件格式
>ImagePolicyWebhook 插件使用了admission config 文件 --admission-control-config-file 来为后端行为设置配置选项
```json
{
  "imagePolicy": {
     "kubeConfigFile": "path/to/kubeconfig/for/backend",
     "allowTTL": 50,           // time in s to cache approval
     "denyTTL": 50,            // time in s to cache denial
     "retryBackoff": 500,      // time in ms to wait between retries
     "defaultAllow": true      // determines behavior if the webhook backend fails
  }
}
```

>这个配置文件必须引用一个 kubeconfig 格式的文件，并在其中配置指向后端的连接，且需要在 TLS 上与后端进行通信
>
>kubeconfig 文件的 cluster 字段需要指向远端服务，user 字段需要包含已返回的授权者

```yaml
# clusters refers to the remote service.
clusters:
- name: name-of-remote-imagepolicy-service
  cluster:
    certificate-authority: /path/to/ca.pem    # CA for verifying the remote service.
    server: https://images.example.com/policy # URL of remote service to query. Must use 'https'.

# users refers to the API server's webhook configuration.
users:
- name: name-of-api-server
  user:
    client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
    client-key: /path/to/key.pem          # key matching the cert
```

### b、请求载荷

>当面对一个准入决策时，API server 发送一个描述操作的 JSON 序列化的 api.imagepolicy.v1alpha1.ImageReview 对象。该对象包含描述被审核容器的字段，以及所有匹配 *.image-policy.k8s.io/* 的 pod 注释。

>注意，webhook API 对象与其他 Kubernetes API 对象一样受制于相同的版本控制兼容性规则。实现者应该知道对 alpha 对象的更宽松的兼容性，并检查请求的 “apiVersion” 字段，以确保正确的反序列化。此外，API server 必须启用 imagepolicy.k8s.io/v1alpha1 API 扩展组 (--runtime-config=imagepolicy.k8s.io/v1alpha1=true)。

* 请求载荷例子：

```json
{  
  "apiVersion":"imagepolicy.k8s.io/v1alpha1",
  "kind":"ImageReview",
  "spec":{  
    "containers":[  
      {  
        "image":"myrepo/myimage:v1"
      },
      {  
        "image":"myrepo/myimage@sha256:beb6bd6a68f114c1dc2ea4b28db81bdf91de202a9014972bec5e4d9171d90ed"
      }
    ],
    "annotations":[  
      "mycluster.image-policy.k8s.io/ticket-1234": "break-glass"
    ],
    "namespace":"mynamespace"
  }
}
```

>远程服务将填充请求的 ImageReviewStatus 字段，并返回允许或不允许访问。响应主体的 “spec” 字段会被忽略，并且可以省略

* 一个允许访问应答会返回：
```
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": true
  }
}
```
* 不允许访问，服务将返回：
```
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": false,
    "reason": "image currently blacklisted"
  }
}
```

