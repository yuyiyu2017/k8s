# 1、滚动升级过程
>旧的pod从副本数减少到0，新的pod从0对应增加到目标数

# 2、针对deploy的升级
* 更新deployment的配置
```
kubectl set image deployment/nginx nginx=nginx:1.12 --record=true
```
>--record=true 会记录更新时的命令

* 查看版本更新状态
```
kubectl rollout status deployment/nginx
```

* 查看版本更新历史
```
kubectl rollout history deployment/nginx
```
* 回滚到上一个版本
```
kubectl rollout undo deployment/nginx
```
>--to-version=3  可以指定回滚到的版本


# 3、针对rc的升级
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


# 4、通过yaml文件完成滚动升级

```
kubectl rolling-update redis-master -f redis-master-controller-v2.yaml
```

* vim redis-master-controller-v2.yaml

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
