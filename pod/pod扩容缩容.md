## 1、动态调整副本命令
```
kubectl scale rc redis-slave --replicas=3
```

## 2、HPA控制器
>根据 CPU 使用情况，预设平局 Pod CPU 使用率，自动调节副本数
>>探测周期为 kube-controller 启动参数 --horizontal-pod-autoscaler-sync-period 定义 (默认为30s)

## 3、示例
>设置 cpu request 为 200m，未设置 limit 上限的值

* vim php-apache-rc.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: php-apache
spec:
  replicas: 1
  template:
    metadata:
      name: php-apache
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: docker.io/googlecontainer/hpa-example
        resources:
          requests:
            cpu: 200m
        ports:
        - containerPort: 80
```

* vim php-apache-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  ports:
  - port: 80
  selector:
    app: php-apache
```

* vim hpa-php-apache.yaml

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: v1
    kind: ReplicationController
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

>规定平均 Pod CPU 使用率维持在 50%
>
>限定副本数量在 1 到 10 之间调整

## 4、查看HPA状态
```
kubectl get hpa
```

## 5、压力测试

```
while true; do wget -q -O- http://php-apache > /dev/null; done
```
> 访问 php-apache 必须要配置 DNS 服务
> 
>查看 rc 状态，副本数变为10
>
>结束循环后，副本数变为1