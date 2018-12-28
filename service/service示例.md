# 一、Service 的类型
>指定 Service 的访问方式，默认为 ClusterIP

* ClusterIP：
>虚拟的服务IP地址，该地址用于集群内部的 Pod 访问，在 Node 上 kube-proxy 通过设置的 iptables 规则进行转发

* NodePort：
>使用宿主机的端口，使能够访问各 Node 的外部客户端通过 Node的 IP 地址和端口号就能访问服务

* LoadBalancer：
>使用外接负载均衡器完成服务的负载分发，需要在 spec.status.loadBalancer 字段指定外部负载均衡器IP， 并同时定义 nodePort 和 clusterIP，用于公有云环境

* ExternalName：
>使用类似DNS域名来匹配service，而不使用标准的selector选择器
>
>通常用于将集群外部的服务引入集群内部

# 二、service中的三层端口

```
ports:
- nodePort: 33062
  port: 88
  protocol: TCP
  targetPort: 80
```
* 容器中的端口
>targetPort: 80

* proxy的端口
>port: 88

* 宿主机的端口
>nodePort: 33062

## 1、创建容器，并指定容器的端口

* vim webapp-rc.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: webapp
spec:
  replicas: 2
  template:
    metadata:
      name: webapp
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx
        ports:
        - containerPort: 80
```

## 2、查看容器的端口
* kubectl describe pod webapp-kctg7
```
Name:               webapp-kctg7
Node:               192.168.112.172/192.168.112.172
IP:                 172.17.59.2
Controlled By:      ReplicationController/webapp
Containers:
  webapp:
    Port:           80/TCP
```

## 3、关联proxy与容器的端口
* 使用yaml文件
>kubectl create -f webapp-svc.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - port: 88
    targetPort: 80
  selector:
    app: webapp
```

* 使用expose命令
>kubectl expose rc webapp
>
>kubectl get svc
>>|NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)|
>>|:--:|:--:|:--:|:--:|:--:|
>>|webapp|ClusterIP|10.0.0.79|<none>|80/TCP|

* 使用expose时指定proxy的端口和service的名称
>kubectl expose rc webapp --port=99 --type=ClusterIP --target-port=80 --name=webapp2
>
>kubectl get svc
>>|NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)|
>>|:--:|:--:|:--:|:--:|:--:|
>>|webapp2|ClusterIP|10.0.0.179|<none>|99/TCP|

## 4、关联宿主机与proxy和容器的端口
* 使用yaml文件
>修改 type 为 NodePort，如果不指定 nodePort: 30003 ，将随机分配端口
>
>如果指定  nodePort: 30003 配置，但 type 为 ClusterIP，将报错
```
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  ports:
  - port: 99
    targetPort: 80
    nodePort: 30003
  selector:
    app: webapp
```

* 使用expose时需指定指定proxy和容器的端口，并指定类型为NodePort 
>kubectl expose rc webapp --port=8081 --type=NodePort --target-port=80 --name=webapp3
>
>kubectl get svc
>>|NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)|
>>|:--:|:--:|:--:|:--:|:--:|
>>|webapp3|NodePort|10.0.0.3|<none>|8081:34098/TCP|

## 5、查看service信息
* kubectl describe svc webapp4 
```bash
Name:                     webapp4
Namespace:                default
Labels:                   app=webapp
Annotations:              <none>
Selector:                 app=webapp
Type:                     NodePort
IP:                       10.0.0.3
Port:                     <unset>  99/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  34098/TCP
Endpoints:                172.17.59.2:80,172.17.76.3:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
* 各种方式访问端口
> 通过容器的IP和端口访问
>> curl 172.17.59.2:80
>>
>>curl 172.17.76.3:80
>
>通过proxy的IP和端口访问
>>curl 10.0.0.3:99
>
>通过宿主机的IP和端口访问
>>curl 127.0.0.1:34098

# 三、负载均衡策略
* RoundRobin
>轮询模式，默认
>
>spec.sessionAffinity: None

* SessionAffinity
>基于客户端 IP 地址进行会话保持的模式
>
>spec.sessionAffinity: ClusterIP

* 自定义负载策略
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx
```
>该 Service 没有虚拟的 ClusterIP 地址
>
>对其进行访问获得具有 Label "app=nginx" 的全部 Pod 列表
>
>然后客户端程序需要实现自己的负载分发策略，来确定访问的后端 Pod


# 四、service多端口和多协议
>对 Service 设置两个端口号，每个端口号进行命名
```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
  - port: 8080
    targetPort: 8080
    name: web
  - port: 8005
    targetPort: 8005
    name: management
  selector:
    app: webapp
```

>两个端口号使用了不同的 4 层协议，即 TCP 或 UDP
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app:kube-dns
  clusterIP: 169.169.0.100
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

# 五、直接绑定宿主机与容器的端口

## 1、将 Pod 端口号映射到物理机
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  containers:
  - name: webapp
    image: tomcat
    ports:
    - containerPort: 8080
      hostPort: 8081
```
>直接访问物理机 IP:8081 即可访问 Pod

## 2、直接使用物理机的网络

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  hostNetwork: true
  containers:
  - name: webapp
    image: tomcat
    imagePullPolicy: Never
    ports:
    - containerPort: 8080
```
>直接访问物理机 IP:8080 即可访问 Pod
>>如果 ports 中不指定 hostPort，默认 hostPort 等于 containerPort
>>
>>如果指定了 hostPort，则 hostPort 必须等于 containerPort


# 六、外部负载均衡器
>通过设置 LoadBalancer 映射到云服务器提供的 LoadBalancer 地址
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - portocol: TCP
    port: 80
    targetPort: 9376
    nodePort: 30061
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155            # 云服务商提供的负载均衡器的 IP 地址
```
>来自外部负载均衡器的流量将直接打到 backend Pod 上，不过实际它们是如何工作的，这要依赖于云提供商。 在这些情况下，将根据用户设置的 loadBalancerIP 来创建负载均衡器。 某些云提供商允许设置 loadBalancerIP。如果没有设置 loadBalancerIP，将会给负载均衡器指派一个临时 IP。 如果设置了 loadBalancerIP，但云提供商并不支持这种特性，那么设置的 loadBalancerIP 值将会被忽略掉

# 七、ExternalName
>某些时候，应用系统需要将一个外部数据库作为后端服务进行连接，或将另一个集群或 Namespace 中的服务作为服务的后端，通过创建一个无 Label Selector 的 Service来实现

* Endpoint方式
>Endpoint IP 地址不能是 loopback（127.0.0.0/8） link-local（169.254.0.0/16） link-local 多播（224.0.0.0/24）
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80                # services 的端口
    targetPort: 9090        # endpoint 的端口
```

>创建一个不带标签选择器的 Service，即无法选择后端的 Pod，系统不会自动创建 Endpoint
>
>手动创建一个和该 Service 同名的 Endpoint，用于指向实际的后端访问地址
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
- addresses:
  - IP: 1.2.3.4             # 外部服务的IP和端口，这里相当于Pod 的IP，只不过是外部的一个不变的地址
  ports:
  - port: 9090
```
>访问没有 selector 的 Service，与有 selector 的 Service 的原理相同。请求将被路由到用户定义的 Endpoint（该示例中为 1.2.3.4:9090）

* 映射域名
>通过CNAME将service（此处my-service）与externalName(my.database.example.com)映射起来
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

# 八、ClusterIP为空
>创建指定ClusterIP为空时，将跳过proxy层的IP，直接网络路径至后端Pod
## 1、创建时的写法
```
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  ports:
  - port: 88
    targetPort: 80
  clusterIP: "None"
  selector:
    app: webapp
```

## 2、查看service信息
>kubectl get svc
>>|NAME|TYPE|CLUSTER-IP|EXTERNAL-IP|PORT(S)|
>>|:--:|:--:|:--:|:--:|:--:|
>>|webapp-svc|ClusterIP|None|<none>|88/TCP|

## 3、解析service地址
>dig -t A webapp-svc.default.svc.cluster.local. @10.0.0.2
>>域名A记录指向所有的后端Pod地址











