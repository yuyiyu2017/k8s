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
>与 Deployment 类似
>
>确保所有或对应标签选择器的Node运行同一个Pod
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

