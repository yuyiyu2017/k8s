# 资源管理

## 一、容器资源限制
### 1、CPU和内存单位
```
CPU：
    1颗逻辑CPU = 1000 millicores
    0.5CPU = 500m

内存：
    Ei、Pi、Ti、Gi、Mi、Ki
```

### 2、资源限制写法
```yaml
spec:
  containers:
    resources:
      requests: 需求，最低保障
      limits: 限制，最大限制
```

### 3、资源限制控制QoS
* 资源限制状态
```tex
# 根据设置的 request 和 limit 决定资源限制状态
# 然后根据资源限制状态，控制pod的存活
pods.status.qosClass
	# 同时设置CPU和内存的requests和limits
	# limits与requests相等
	Guranteed:
	# 至少有一个容器设置CPU或内存资源的requests属性
	Burstable:
	# 没有设置过资源限制
	BeastEffort:
```

* 存活规则
```tex
当资源不足时，先干掉BeastEffort，再干掉Burstable，最后比较Guranteed
已占用资源/requests资源 比值越大，越容易被干掉
```

### 4、资源limits演示

* limit_test1.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: lorel/docker-stress-ng
    command: ["/usr/bin/stress-ng", "-m 1", "-c 1", "--metrics-brief"]
    resources:
      requests:
        cpu: "200m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

* 解释说明
```tex
"/usr/bin/stress-ng", "-m 1", "-c 1", "--metrics-brief"
-m 对memory启动一个进程压测
-c 对CPU启动一个进程压测
```

* 现象结果
```bash
docker stats CONTAINER_ID
	创建后，进入所在Node上，查看容器状态
	如果所在Node有2核，则会显示cpu消耗为25%
```

## 二、资源限制规则

### 1、LimitRanges
> 官方文档 https://kubernetes.io/docs/concepts/policy/limit-range/
> 
> 对namespace设置默认资源限制

* 书写格式
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: 此规则的名称
  namespace: 指定名称空间进行限制
spec:
  limits:
  - default: <map[string]string>
        Pod/Container 默认的资源 limits

    defaultRequest: <map[string]string>
        Pod/Container 默认的资源 requests

    max: <map[string]string>
        Pod/Container 的 limits 的最大值
        PersistentVolumeClaim 申请容量的最大值

    maxLimitRequestRatio: <map[string]string>
        Pod/Container 的 limits 最少大于 request 的倍数

    min: <map[string]string>
        Pod/Container 的 limits 的最小值
        PersistentVolumeClaim 申请容量的最小值

    type: <string>
        此资源限制应用到的Kind
        可为 Pod/Container/PersistentVolumeClaim
        针对不同的 Kind，其他参数的设置将有所区别
```

### 2、ResourceQuotas
> 官方文档 https://kubernetes.io/docs/concepts/policy/resource-quotas/
> 
> 对namespace设置资源配额，用来限制namespace中的总资源

* 书写格式
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: 规则名称
  namespace: 指定名称空间进行配额
spec:
  hard: <map[string]string>
    限制的对象

  scopeSelector: <Object>
    In
    NotIn
    Exist
    DoesNotExist

  scopes: <[]string>
    直接匹配应用对象，scopeSelector为范围匹配
```

* 计算资源配额示例
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: spark-cluster
spec:
  hard:
    pods: "20"
    requests.cpu: "20"
    requests.memory: 100Gi
    limits.cpu: "40"
    limits.memory: 200Gi
    requests.nvidia.com/gpu: 4
```

* 对象数量限制示例
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: spark-cluster
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
```

* namespace中所有资源数量都可限制

```bash
# 结构都为 count/<resource>.<group>:
kubectl create quota test \
--hard=count/deployments.extensions=2,count/replicasets.extensions=4,count/pods=3,count/secrets=4 \
--namespace=myspace
```

### 3、ResourceQuotas与PriorityClass的结合
> 设置多个PriorityClass，对应不同的ResourceQuotas
> 
> 创建Pod时，可以引用PriorityClass，从而应用对应的ResourceQuotas

* quota.yaml
```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
```

* high-priority-pod.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

## 三、Pod优先级与抢占

### 1、设置过程
```tex
1、定义PriorityClass，不同PriorityClass的value不同，value越大优先级越高

2、创建Pod，并设置Pod的priorityClassName字段为指定的PriorityClass

3、当资源限制时，优先级高的Pod会抢占更多资源
```

### 2、默认的PriorityClass
* kubectl get  priorityclass
```tex
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            18h
system-node-critical      2000001000   false            18h
```

### 3、PriorityClass书写格式
```yaml
apiVersion: scheduling.k8s.io/v1
description: <string>
kind: PriorityClass
metadata:
  name: <string>
value: <integer>
globalDefault: <boolean>
	是否设置为全局的默认PriorityClass，只能有一个PriorityClass应用为全局默认
preemptionPolicy: <string>
	测试a阶段
	当设置为Never时，对应Pod不会抢占优先级比其低的Pod，而只会创建时优先被分配调度
```



