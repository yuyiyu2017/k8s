## 1、搭建方法
* 下载yaml
```
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/coredns/coredns.yaml.base
```

* 修改配置
```
clusterIP: __PILLAR__DNS__SERVER__
    修改为 
        clusterIP: 10.0.0.2
    对应kubelet配置中的clusterDNS: ["10.0.0.2"]
    DNS服务在集群中的proxy层的IP

kubernetes __PILLAR__DNS__DOMAIN__ in-addr.arpa ip6.arpa
    修改为 
        kubernetes cluster.local. in-addr.arpa ip6.arpa
    对应kubelet配置中的clusterDomain: cluster.local.

image: k8s.gcr.io/coredns:1.3.1
    修改镜像为可用镜像 
        image: yuyiyu/google-containers.coredns:1.3.1

memory: __PILLAR__DNS__MEMORY__LIMIT__
    修改为 memory: 100Mi
    设置分配给DNS服务的内存

修改副本数，保证高可用
    新版本中副本数由k8s的自动伸缩管理
    也可以自行设置副本数 replicas: 2
```

* 创建服务
```
kubectl apply -f coredns.yaml
```

## 2、检查是否生效
* 创建测试busybox
```
kubectl apply -f busybox.yaml 
```

* 默认空间
```bash
kubectl exec busybox -- nslookup kubernetes
```

* 指定空间
```bash
kubectl exec busybox -- nslookup kubernetes-dashboard.kube-system
kubectl exec busybox -- nslookup 服务名.空间名
```

## 3、每条dns都会记录在etcd中
* 查看etcd所有键
```
export ETCDCTL_API=3
etcdctl --cacert=/data/k8s/etcd/ssl/ca.pem --cert=/data/k8s/etcd/ssl/server.pem --key=/data/k8s/etcd/ssl/server-key.pem --endpoints="https://192.168.112.171:2379,https://192.168.112.172:2379,https://192.168.112.173:2379" get / --prefix --keys-only
```

## 4、Serviceaccount说明
> 使用DNS的前提，必须启用kube-apiserver的token认证

* 创建证书并启动参数后重启服务
```
apiserver的启动参数中添加：
    --enable-admission-plugins=ServiceAccount
    --token-auth-file=/data/k8s/master/cfg/token.csv
    --service-account-key-file=/data/k8s/master/ssl/ca-key.pem

kube-controller-manager
    --service-account-private-key-file=/data/k8s/master/ssl/ca-key.pem
```

* 查看当前的 ServiceAccount
```
# 必须有一个
kubectl get serviceaccount
```



