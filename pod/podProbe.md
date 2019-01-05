# 一、pod生命周期和重启策略

## 1、Pod的状态

>Pending
>>API Server 已经创建该Pod，但Pod内还有一个或多个容器的镜像没有创建，包括正在下载镜像的过程
>
>Running
>>Pod 内所有容器均已创建，且至少有一个容器处于运行状态、正在启动状态或正在重启状态
>
>Succeeded
>>Pod 内所有容器均成功执行退出，且不会再重启
>
>Failed
>>Pod 内所有容器均已退出，但至少有一个容器退出为失败状态
>
>Unknown
>>由于某种原因无法获取该Pod的状态，可能由于网络通信不畅导致


## 2、Pod重启策略     RestartPolicy
>Always：
>>Pod一旦终止运行，则无论容器是如何终止的，kubelet都将重启它
>
>OnFailure：
>>只有Pod以非零退出码终止时，kubelet才会重启该Pod
>
>Never：
>>Pod终止后，kubelet将退出码报告给Master，不会再重启该Pod

## 3、可用于管理Pod的控制器
>RC 和 DaemonSet：
>>必须设置为Always，需要保证该容器持续运行
>
>Job：
>>OnFailure 或 Never，确保容器执行完成后不再重启
>
>kubelet：
>>在Pod失效时自动重启它，不论 RestartPolicy 设置的值，并且也不会对Pod进行健康检查



# 二、Pod健康检查
## 1、探针类型
* LivenessProbe
>用于判断容器是否存活(running状态)
>
>如果探测到容器不健康，kubelet将杀掉该容器，并根据重启策略对应处理
>
>如果容器不包含 LivenessProbe探针，kubelet将认为容器的返回值永远时"Success"

* ReadinessProbe
>用于判断容器是否完成(ready状态)
>
>如果检测到失败，则 Pod 的状态将被修改
>
>Endpoint Controller 将从 Service 的 Endpoint 中删除包含该容器所在 Pod 的 Endpoint

## 2、ExecAction探测方式

> 在容器内部执行一个命令，如果命令返回码为0，表明容器健康

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: gcr.io/google_containers/busybox
    args:
    - /bin/sh
    - -c
    - echo ok > /tmp/health; sleep 10; rm -rf /tmp/health; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/health
      initialDelaySeconds: 15
      timeoutSeconds: 1
```

>通过执行 cat /tmp/health ，判断容器运行是否正常:
>>该Pod运行后，创建 /tmp/health ，10秒后删除，初始探测时间为15秒，探测结果是 Fail ，导致kubelet杀掉该容器并重启Pod


## 3、TCPSocketAction探测方式

> 通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 30
      timeoutSeconds: 1
```

> 通过与容器内 localhost:80 建立TCP连接，判断容器健康


## 4、HTTPGetAction探测方式

> 通过容器的IP地址、端口号及路径调用HTTP Get方法，如果响应的状态码大于等于 200 且小于等于 400，则表明容器健康

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /_status/healthz
        port: 80
      initialDelaySeconds: 30
      timeoutSeconds: 1
```

> 通过访问容器的 localhost:80/_status/healthz ，判断容器健康