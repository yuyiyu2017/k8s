## 一、各服务介绍
```
master
    etcd
        分布式键值存储系统，用于保存集群状态数据
    kubectl
        管理工具
    apiserver
        api接口
    scheduler
        根据调度算法为新建的Pod选择Node
    controller-manager
        处理集群中后台任务，管理各种资源控制器

node
    kubelet
        接收master的指令，管理本node上的资源
    kube-proxy
        对Node进行负载均衡和网络规则转发

pause
    k8s的注册服务，每个容器都会有一个对应的pause容器对应
```


```
Pod
    最小部署单元
    一组容器的集合
    一个Pod中的容器共享网络命名空间

Controllers
    ReplicaSet
        确保预期的Pod副本数量

    Deployment
        无状态应用部署

    StatefulSet
        有状态应用部署

    DaemonSet
        确保所有Node运行同一个Pod

    Job
        一次性任务

    Cronjob
        定时任务

Service
    防止Pod失联
    定义一组Pod的访问策略

Label
    标签，用于关联对象、查询和筛选

Namespaces
    命名空间，将对象逻辑上隔离

Annotations
    注释，用于运维和开发查看提示
```

## 二、kubectl命令补充
```
查看资源对象的详细说明
    kubectl explain 

查看pods的yaml文件含义
    kubectl explain pods

然后可以一级一级地查看
    kubectl explain pods.status
    kubectl explain pods.status.phase

查看资源的标签信息
    kubectl get pods --show-labels

通过标签匹配资源
    kubectl get pods -l run=nginx

查看资源详细信息
    kubectl get pods -o wide

以json形式查看配置
    kubectl get pods busybox -o json
```

## 三、日志排错
```
查看 Pod 的日志
    kubectl logs -f pods PODNAME -n NAMESPACE

如果一个Pod中有多个服务
    kubectl logs -f pods PODNAME -c ONE_SERVICE -n NAMESPACE

查看集群系统状态
    kubectl cluster-info
```

## 四、swarm 创建和使用网络
```
docker network create --driver overlay --opt encrypted --subnet 10.10.110.0/24 sitewhere
docker service create --replicas 5 --network sitewhere --name my-test -p 80:80 nginx
```

## 五、yaml文件意义
```tex
1、定义配置时，指定最新稳定版API（当前为v1）
2、配置文件应该存储在集群之外的版本控制仓库中。如果需要，可以快速回滚配置、重新创建和恢复
3、应该使用YAML格式，而不是JSON
4、可以将相关对象组合成单个文件，更容易管理
5、不要没必要的指定默认值，简单和最小配置，从而减少错误
6、在注释中说明，更好维护
```

## 六、查看所有的API版本
```
kubectl api-versions
```
```tex
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
crd.projectcalico.org/v1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```


