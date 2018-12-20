## 1、搭建方法
* kubectl create -f coredns.yaml
>修改dns服务在集群中的proxy层IP
>>clusterIP: 10.0.0.2
>>
>>对应kubelet配置中的clusterDNS: ["10.0.0.2"]

>修改dns服务在集群中的域名
>>kubernetes cluster.local. in-addr.arpa ip6.arpa
>>
>>对应kubelet配置中的clusterDomain: cluster.local.

>修改镜像为可用镜像
>>image: coredns/coredns:1.0.6

>
>修改副本数，保证高可用
>>replicas: 2

## 2、检查是否生效
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
