# Pod调度

>Pod通常只是容器的载体，需要通过 RC、Deployment、DaemonSet、Job等完成Pod的调度与自动控制功能

## 一、RC、Deployment：全自动调度

>自动部署一个容器应用的多份副本，以及持续监控副本的数量

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  replicas: 3
  selector:
    name: frontend
  template:
    metadata:
      labels:
        name: frontend
    spec:
      containers:
      - name: frontend
        image: kubeguide/guestbook-php-frontend
        env:
        - name: GET_HOSTS_FROM
          value: env
        ports:
        - containerPort: 80
```

>指定类型为RC管理
>>kind: ReplicationController
>
>指定副本数量始终为3
>>spec.replicas: 3


### 1、RC下的 NodeSelector 定向调度

>通过给各 Node 打标签，创建 Pod 时指定 nodeSelector，使容器落在指定的 Node 上

* 给节点打上标签
```
kubectl label nodes <node-name> <label-key>=<label-value>
```

* yaml中指定 nodeName 和 nodeSelector

```
kubectl label node 192.168.112.173 abc=def3
```

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  replicas: 1
  selector:
    name: redis-master
  template:
    metadata:
      labels:
        name: redis-master
    spec:
      containers:
      - name: master
        image: docker.io/kubeguide/redis-master
        ports:
        - containerPort: 6379
      nodeName: node01
      nodeSelector:
        abc:def3
```

>如果多个 Node 有相同的标签，则 scheduler 会根据调度算法从中选取一个
>
>如果指定 nodeSelector ，且集群中没有任何对应标签的 Node，则创建失败


### 2、RC下的 NodeAffinity 亲和性调度

* NodeAffinity操作符
>In \ NotIn \ Exists \ DoesNotExist \ Gt \ Lt

* NodeAffinity策略
>RequireDuringSchedulingRequiredDuringExecution
>>与 NodeSelector 类似，但当 Node 不满足条件时，系统将从该 Node 上移除之前调度上的 Pod
>
>RequireDuringSchedulingIgnoredDuringExecution
>>与上一个策略相似，但当 Node 不满足条件时，系统不一定从该 Node 上移除之前调度上的 Pod
>
>PreferredDuringSchedulingIgnoredDuringExecution
>>指定在满足调度条件的 Node 中，那些 Node 应更优先地进行调度
>>
>>当 Node 不满足条件时，系统不一定从该 Node 上移除之前调度上的 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-labels
  annotations:
    scheduler.alpha.kubernetes.io/affinity: >
      {
        "nodeAffinity": {
          "requireDuringSchedulingIgnoredDuringExecution": {
            "nodeSelectorTerms": [
              {
                "matchExpressions": [
                  {
                    "key": "kubernetes.io/e2e-az-name",
                    "operator": "In",
                    "values": ["e2e-az1", "e2e-az2"]
                  }
                ]
              }
            ]
          }
        }
      }
    another-annotation-key: another-annotation-value
spec:
  contaioners:
  - name: with-labels
    image: gcr.io/google_containers/pause:2.0
```

>只有 Node 的 Label 中包含 key=kubernetes.io/e2e-az-name，且其 value 为 "e2e-az1" 或 "e2e-az2" 时，才能成为该Pod的调度目标




## 二、DaemonSet：特定场景调度

>用于管理在集群中每个 Node 上仅运行一份 Pod 的副本实例

* 应用场景
>在每个Node上运行一个 GlusterFS 存储或者 Ceph 存储的 daemon 进程
>
>在每个Node上运行一个日志采集系统，如 logstash


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
>每个 Node 上都有一个 fluentd 容器

## 三、Job：批处理调度，目前系统并不完善

### 1、四种模式

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

### 2、三种类型

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


### 3、Job Template Expansion模式

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