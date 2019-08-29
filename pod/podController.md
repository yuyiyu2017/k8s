# 一、Controllers类型
>Pod副本控制器，控制Pod的创建、状态及回收

* Replication Controllers
>早期的副本控制器

* ReplicaSet
>替代Replication Controllers的新版，支持更多的selector特性

* Deployment
>无状态应用部署，联系多个 ReplicaSet，实现版本切换

* StatefulSet
>有状态应用部署

* DaemonSet
>与 Deployment 类似，确保所有或标签对应Node都运行同一个Pod

* Job
>一次性任务
>>正确完成后就结束，如果没完成将持续进行
>>
>>比如Pod创建失败，任务没完成，将重新创建Pod

* Cronjob
>周期定时任务

# 二、ReplicaSet
>通过关联对应的Label控制Pod数量
>>一个标签选择器，选择目标Pod
>>
>>一个数字，表明目标Pod的期望实例数
>>
>>一个Pod模板，用来创建Pod实例

* 查看解释
```
kubectl explain rs 
```
```
KIND:     ReplicaSet
VERSION:  extensions/v1beta1

FIELDS:
    apiVersion      <string>        # 与 VERSION 对应，extensions/v1beta1
    kind            <string>        # 与 TYPE 对应，ReplicaSet
    metadata        <Object>        # 元数据
    spec            <Object>
        replicas        <integer>               # 副本数
        selector        <map[string]string>     # 标签选择器
        template        <Object>                # 嵌套Pod模板
    status          <Object>        # 只读的状态
```

# 三、Deployment
>无状态应用部署
>
>联系多个 ReplicaSet，实现版本切换
>
>每次更新会生成一个新的 ReplicaSet
>
>只有一个 ReplicaSet 活动，默认保留10个历史版本
## 1、查看解释

```
kubectl explain deploy
```
```
KIND:     ReplicaSet
VERSION:  extensions/v1beta1

FIELDS:
    apiVersion      <string>        # 与 VERSION 对应，extensions/v1beta1
    kind            <string>        # 与 TYPE 对应，ReplicaSet
    metadata        <Object>        # 元数据
    spec            <Object>
        replicas        <integer>               # 副本数
        selector        <map[string]string>     # 标签选择器
        template        <Object>                # 嵌套Pod模板
        strategy        <Object>                # 更新策略
    status          <Object>        # 只读的状态
```

## 2、apply申明式创建
```
kubectl apply -f deploy-demo.yaml
```
>创建deployment，会同步创建与deployment同名加其哈希值为名的RS
>>DEPLOYNAME-HASH

>然后创建的Pod名称为RS加随机哈希值
>>DEPLOYNAME-HASH-RANDOMHASH

>每次apply，会生成新的RS版本

## 3、可以控制副本更新过程
```
spec.strategy
```
* 滚动发布
>增加一个，删除一个，交替进行直到全部更新

* 金丝雀/灰度发布
>先更新一部分，测试不通过则回滚，测试无误则推广到全局

* 蓝绿发布
>先增加相同数量的新版本，测试后，全部完整替换原版本

## 4、更新版本
>更新命令适用与其他资源，如 Pod、Service 等
* set
>会触发版本变更，创建新的RS
```bash
kubectl set image deployment/nginx nginx=nginx:1.12 --record=true
    --record=true 会记录更新时的命令
```
```
env            Update environment variables on a pod template
image          更新一个 pod template 的镜像
resources      在对象的 pod templates 上更新资源的 requests/limits
selector       设置 resource 的 selector
serviceaccount Update ServiceAccount of a resource
subject        Update User, Group or ServiceAccount in a RoleBinding/ClusterRoleBinding
```

* patch
>不会触发版本变更，从而不会创建新的RS
```bash
kubectl patch deployment/nginx -p '{"spec":{"replicas":5}}'
	-p 指定对应字段的json
```

* rollout
```
# 查看版本更新状态
kubectl rollout status deployment/nginx

# 查看版本更新历史
kubectl rollout history deployment/nginx

# 回滚到上一个版本
kubectl rollout undo deployment/nginx
    --to-version=3 可以指定回滚到的版本
```

# 四、DaemonSet
> 与 Deployment 类似
>
> 确保所有或对应标签选择器的Node运行同一个Pod
```
KIND:     DaemonSet
VERSION:  extensions/v1beta1

FIELDS:
    apiVersion      <string>        # 与 VERSION 对应，extensions/v1beta1
    kind            <string>        # 与 TYPE 对应，ReplicaSet
    metadata        <Object>        # 元数据
    spec            <Object>
        selector        <map[string]string>     # 标签选择器
        template        <Object>                # 嵌套Pod模板
        strategy        <Object>                # 更新策略
    status          <Object>        # 只读的状态
```

* 应用场景
> 在每个Node上运行一个 GlusterFS 存储或者 Ceph 存储的 daemon 进程
>
> 在每个Node上运行一个日志采集系统，如 logstash

* vim fluentd-ds.yaml

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd-cloud-logging
  namespace: kube-system
  labels:
    k8s-app: fluentd-cloud-logging
spec:
  template:
    metadata:
      namespace: kube-system
      labels:
        k8s-app: fluentd-cloud-logging
    spec:
      containers:
      - name: fluentd-cloud-logging
        image: docker.io/googlecontainer/fluentd-elasticsearch:1.17
        resources:
          limits:
            cpu: 100m
            memory: 200Mi
        env:
        - name: FLUENTD_ARGS
          value: -q
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: false
        - name: containers
          mountPath: /var/lib/docker/containers
          readOnly: false
      volumes:
      - name: containers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
```

* 创建 DaemonSet
```
kubectl create -f fluentd-ds.yaml
```

* 查看 Pod

```
kubectl get pods --namespace=kube-system
```

# 五、旧版rc的升级
* 将 redis-master 从 1.0 提升为 2.0
```
kubectl rolling-update redis-master --image=redis-master:2.0
```
* 更新完成后，查看 RC
```
kubectl get rc
```
> kubectl会给 RC 增加一个 key 为 "deployment" 的标签，其值为 hash 计算后的值
>> 可通过 --deployment-label-key 参数修改 key 名

* 如果中途发现配置有误，可以中断更新，并回滚
```
kubectl rolling-update-rollback
```
>新的 RC 将使用旧的 RC 的名字

* 通过yaml文件完成滚动升级

```
kubectl rolling-update redis-master -f redis-master-controller-v2.yaml
```

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master-v2
  labels:
    name: redis-master
    version: v2
spec:
  replicas: 1
  selector:
    name: redis-master
    version: v2
  template:
    metadata:
      labels:
        name: redis-master
        version: v2
    spec:
      containers:
      - name: master
        image: kubeguide/redis-master:2.0
        ports:
        - containerPort: 6379
```

>RC 的名字(name)不能与旧的 RC 的名字相同:
>>name: redis-master-v2

>在 selector 中至少有一个Label与旧的 RC 的 Label 不同，以标识其为新的 RC
>>version: v2

# 六、StatefulSet
## 1、有状态服务要求

```
a、稳定且唯一的网络标识符
b、稳定且持久的存储
c、有序、平滑地部署和扩展
d、有序、平滑地删除和终止
e、有序地滚动更新
```

## 2、实现需要地三个组件
```
headless service
StatefulSet
volumeClaimTemplate
```

## 3、StatefulSet使用例举

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  labels:
    app: myapp-svc
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: myapp-pod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  serviceName: myapp-svc
  replicas: 3
  selector:
    matchLabels:
      app: myapp-pod
  template:
    metadata:
      labels:
        app: myapp-pod
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v5
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: myappdata
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: myappdata
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs-client"
      resources:
        requests:
          storage: 2Gi
```

## 4、使用解释

* 一个headless的service

```
# kubectl get svc

myapp-svc   ClusterIP   None   <none>  80/TCP  13m
```

* Pod名称规则
>pod名称与containers的name对应，按顺序添加后缀
```
# kubectl get pods

myapp-0
myapp-1
myapp-2
```

* Pod访问方式
>每个pod会被单独分配域名与之对应
>
>分配的域名格式
>>pod_name.service_name.namespace_name.svc.cluster.local.
```
# kubectl exec -it myapp-0 -- /bin/sh

/ # nslookup myapp-0

Name:      myapp-0
Address 1: 172.17.99.5 myapp-0.myapp-svc.default.svc.cluster.local.
```

* Pod对应的PVC
>每个pod对应固定名称的pvc
>
>名称格式为
>>volumeClaim_name-pod_name

```
# kubectl get pvc

myappdata-myapp-0
myappdata-myapp-1
myappdata-myapp-2
```

## 5、StatefulSet滚动更新配置

>当partition为N时，Pod尾号大于等于N的更新，默认为0
>
>通过调整N的值，动态控制更新过程
```yaml
spec:
  updateStrategy:
    type:
      rollingUpdate:
        partition: Int
```

# 七、Job批处理调度(需要时研究)

## 1、四种模式

>Job Template Expansion模式
>>一个 Job 对象对应一个待处理的 Work item
>>
>>有几个 Work item 就产生几个独立的 Job
>>
>>适用 Work item 数量少、每个 Work item 要处理的数据量比较大的场景

>Queue with Pod Per Work Item模式
>>采用一个任务队列存放 Work item
>>
>>一个 Job 对象作为消费者逐个完成这些 Work item
>>
>>Job 会启动 N 个 Pod，每个 Pod 对应一个 Work item

>Queue with Variable Pod Count模式
>>采用一个任务队列存放 Work item
>>
>>一个 Job 对象作为消费者逐个完成这些 Work item
>>
>>但 Job 启动的 Pod 数量是可变的

>Single Job with Static Work Assignment模式
>>采用程序静态方式分配任务
>>
>>一个 Job 对象产生多个 Pod

## 2、三种类型

>Non-parallel Jobs
>>一个 Job 只启动一个 Pod
>>
>>除非 Pod 异常，才会重启该 Pod
>>
>>一旦 Pod 正常结束，Job将结束

>Parallel Jobs with a fixed completion count
>>并行 Job 会启动多个 Pod
>>
>>需要设定 Job 的 spec.completions 参数为一个正数
>>
>>当正常结束的 Pod 数量达到此参数的设定值时，Job 结束
>>
>>Job 的 spec.parallelism 参数控制并行度，即同时启动几个 Job 处理 Work item

>Parallel Jobs with a work queue
>>Work item 都在一个 Queue 中存放，不能设置 spec.completions 参数
>>
>>每个 Pod 能独立判断和决定是否还有任务项需要处理
>>
>>如果某个 Pod 正常结束，则 Job 不会再启动新的 Pod
>>
>>如果一个 Pod 成功结束，则其他 Pod 应都处于即将结束或退出的状态
>>
>>如果所有 Pod 都结束了，且至少一个 Pod 成功结束，则整个 Job 算成功结束


## 3、Job Template Expansion模式

* vim job.yaml

```
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

* vim 123.sh

```
for i in apple banana cherry
do
  cat job.yaml |sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
```

* 创建 Job
```
kubectl create -f jobs
```

* 查看运行状态

```
kubectl get jobs
```

```
process-item-apple    1         1            4m
process-item-banana   1         1            4m
process-item-cherry   1         1            4m
```




