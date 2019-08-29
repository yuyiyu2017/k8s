# Pod调度

## 一、 调度策略

### 1、Scheduler分配Pod到Node的过程
```tex
预选 Predicate
    排除完全无法运行的节点
	https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/algorithm/predicates/predicates.go

优选 Priority
    按一定规则进行优先级计算和排序
    https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/algorithm/priorities

随机选择 Select
```

### 2、预选策略:

```tex
CheckNodeCondition:

GeneralPredicates:
    HostName: 检查Pod对象是否定义了pod.spec.hostname
    PodFitsHostPorts: pods.spec.containers.ports.hostPort
    MatchNodeSelector: pods.spec.nodeSelector
    PodFitsResources: 检查节点是否能满足Pod的资源需求

NoDiskConflict: 
	检查节点是否满足Pod依赖的存储卷

PodToleratesNodeTaints: 
	检查Pod上的 spec.tolerations 可容忍的污点，是否完全包含节点上的污点。
	只针对创建Pod时的污点比较，已经创建完成后，如果出现新的污点，Pod将会忽略

# 默认不启用的
PodToleratesNodeNoExecuteTaints: 
	检查节点上的污点状态，如果运行中节点的污点不符合了，Pod将离开此节点。
CheckNodeLabelPresence: 
	检查标签的存在性
CheckServiceAffinity: 
	将Pod调度到与其在同一Service的Pod所在的节点

# 云存储的挂载限制
MaxEBSVolumeCount: Aws
MaxGCEPDVolumeCount: Google
MaxAzureDiskVolumeCount: Azure

# 存储限制
CheckVolumeBinding: 
	检查节点是否有合适Pod的PVC
NoVolumeZoneConflict: 
	检查Pod所需存储与节点所在区域是否冲突

# 检查节点硬件的压力状态
CheckNodeMemoryPressure
CheckNodeDiskPressure
CheckNodePIDPressure

MatchInterPodAffinity: 
	检查自定义的亲和策略
```

### 3、优选策略
```tex
LeastRequested: 
    剩余资源占所有资源比值越高，优先级越高
    (cpu((capacity-sum(requested))*10/capacity) + memory((capacity-sum(requested))*10/capacity))/2

    一个节点剩余16核32G
    Pod请求的资源为4核8G
    则得到此节点的优先级为 ((16-4)*10/16 + (32-8)*10/32)/2 = 7.5

BalancedResourceAllocation:
    CPU和内存资源被占用率相近，优先级越高
    当CPU利用了60%，内存利用率越接近60%，优先级越高

NodePreferAvoidPods:
    节点的注解信息anonation 是否存在scheduler.alpha.kubernetes.io/preferAvoidPods
    如果存在注解，得分直接为0，如果不存在注解，得分直接为10

TaintToleration:
    将Pod的spec.tolerations列表项与节点的taints列表项进行匹配度检查
    匹配的条目越多，得分越低

SelectorSpreading:
    尽量将相同的标签分散开来
    如果某个节点上有与当前Pod相同的标签选择器的Pod越多，得分越低

InterPodAffinity:
    Pod亲和性比较
    与Pod的亲和性条目项越多，得分越高

NodeAffinity
    节点亲和性比较

# 后三个默认不启用
MostRequested:
    尽量让Pod在使用较多资源的节点上运行
    与LeastRequested相反，不能同时使用

NodeLabel:
    比较节点的标签
    标签存在越多，优先级越高

ImageLocality:
    将Pod尽量分配到满足需求的已有镜像的节点上
    不是根据镜像数量，而是与镜像的体积大小之和比较
```

## 二、三类标识
```tex
Label

Annonation

Tolerations：定义在Pod之上
Taints：定义在Node之上
```

## 三、节点选择器
 * nodeSelector
 * nodeName

## 四、亲和性affinity

### 1、在annotations中设置亲和性
> 写法与在spec.affinity设置类似
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-labels
  annotations:
    scheduler.alpha.kubernetes.io/affinity:
      nodeAffinity:
        requireDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: "kubernetes.io/e2e-az-name"
              operator: "In",
              values: ["e2e-az1", "e2e-az2"]
    another-annotation-key: another-annotation-value
spec:
  contaioners:
  - name: with-labels
    image: gcr.io/google_containers/pause:2.0
```

### 2、在spec.affinity设置亲和性
```yaml
spec:
  affinity:
    nodeAffinity:       Pod与Node的比较判断亲和性
    podAffinity:        当前Pod与其他Pod比较，让一组Pod尽量在一起
    podAntiAffinity:    当前Pod与其他Pod比较，让不同组的Pod尽量不在一起
```

### 3、nodeAffinity的prefer示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-labels
  labels:
    app: myapp
    tier: frontend
spec:
  contaioners:
  - name: myapp
    image: ikubernetes/myapp:v1
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - preference	
            matchExpressions
            - key: "kubernetes.io/e2e-az-name"
              operator: "In",
              values:
              - e2e-az1
              - e2e-az2
        weight: 60
```

### 4、nodeAffinity的require示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-labels
  labels:
    app: myapp
    tier: frontend
spec:
  contaioners:
  - name: myapp
    image: ikubernetes/myapp:v1
  affinity:
    nodeAffinity:
      requireDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: "kubernetes.io/e2e-az-name"
            operator: "In",
            values:
            - e2e-az1
            - e2e-az2
```

### 5、podAffinity示例
> pod-second会更倾向到，与pod-first所在Node的label中，键kubernetes.io/hostname的值相同的Node上
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-first
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-second
  labels:
    app: backend
    tier: db
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh","-c","sleep 3600"]
  affinity:
    podAffinity:
      requireDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - {key: app, operator: In, values: ["myapp"]}
        topologKey: kubernetes.io/hostname
```

## 五、Node的taints污点
### 1、yaml写法
* kubectl explain nodes.spec.taints
```tex
effect	<string> -required-
    定义对Pod的排斥效果:
        NoSchedule:
            仅影响调度过程，对现存的Pod对象不产生影响
            Pod对象创建时会考虑，后续添加的taints不产生影响
        NoExecute:
            不仅影响调度过程，也对现存的Pod对象产生影响
            后续添加的taints，会将Pod驱离
        PreferNoSchedule:
            倾向性的污点，影响亲和性算法
key	<string> -required-

value	<string>
```

### 2、taints命令

* 查看节点taints
```bash
kubectl describe node [node]
```

* 为节点设置taints
```bash
kubectl taint node [node] key1=value1:NoSchedule
kubectl taint node [node] key1=value1:NoExecute
kubectl taint node [node] key1=value1:NoSchedule
```

* 取消节点的taints
```bash
# 在后面添加'减号'，可以只指定key
kubectl taint node [node] key1-
kubectl taint node [node] key1=value1-
```

### 3、taints示例
* 查看节点的taints
```bash
# kubeadm安装的master节点是有污点的，普通服务是不会在master上创建pod
kubectl describe node centos3
	Taints: node-role.kubernetes.io/master:NoSchedule
```

* 为节点设置污点
```bash
# 二进制安装的节点没有设置taints，可以添加
kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule
```

* 取消节点的污点
```bash
kubectl taint node centos3 node-role.kubernetes.io/master:NoSchedule-
```

### 4、Pod的tolerations
> 设置过tolerations的Pod会忽略node的taints

* kubectl explain pods.spec.tolerations

```tex
effect	<string>
key	<string>
operator	<string>
tolerationSeconds	<integer>
value	<string>
```

* 示例，让pod可以调度到master
```yaml
spec:
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Equal"
    value: ""
    effect: "NoSchedule"
```

