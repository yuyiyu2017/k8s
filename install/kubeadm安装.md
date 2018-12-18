# 一、各节点安装
## 1、主机名文件
* vim /etc/hosts
```
10.27.100.242 ubuntu62-virtual-machine
10.27.100.243 ubuntu63-virtual-machine
10.27.100.244 ubuntu64-virtual-machine
```

## 2、阿里云k8s仓库

```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
```

## 3、安装k8s三组件
```
apt-get install -y kubelet kubeadm kubectl
```

# 二、其他准备

## 1、临时关闭swap
```
swapoff -a
```

## 2、永久关闭swap
```
注释 /etc/fstab
```

## 3、防火墙开放
```bash
iptables -P FORWARD ACCEPT
```

## 4、启用ipvs模块
* vim /etc/rc.local
```bash
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe ip_vs
```

# 三、将谷歌镜像拉取k8s的包并tag
>必须要指定版本，这样kubeadm才不会去连k8s.io，kubeadm init同理

## 1、首先查询下当前版本需要哪些docker image

>k8s镜像的版本必须不低于 kubectl 的版本
>>kubectl version

```
kubeadm config images list --kubernetes-version v1.13.0
```

>k8s.gcr.io/kube-apiserver:v1.13.0
>k8s.gcr.io/kube-controller-manager:v1.13.0
>k8s.gcr.io/kube-scheduler:v1.13.0
>k8s.gcr.io/kube-proxy:v1.13.0
>k8s.gcr.io/pause:3.1
>k8s.gcr.io/etcd:3.2.24
>k8s.gcr.io/coredns:1.2.6

>如果不加 --kubernetes-version 将显示最新版本



## 2、在国外服务器上拉取谷歌镜像，然后推送到国内hub.docker

>docker pull k8s.gcr.io/kube-apiserver:v1.13.0
>docker pull k8s.gcr.io/kube-controller-manager:v1.13.0
>docker pull k8s.gcr.io/kube-scheduler:v1.13.0
>docker pull k8s.gcr.io/kube-proxy:v1.13.0
>docker pull k8s.gcr.io/pause:3.1
>docker pull k8s.gcr.io/etcd:3.2.24
>docker pull k8s.gcr.io/coredns:1.2.6
>
>docker tag k8s.gcr.io/kube-apiserver:v1.13.0 yuyiyu/google-containers.kube-apiserver:v1.13.0 
>docker tag k8s.gcr.io/kube-controller-manager:v1.13.0 yuyiyu/google-containers.kube-controller-manager:v1.13.0
>docker tag k8s.gcr.io/kube-scheduler:v1.13.0 yuyiyu/google-containers.kube-scheduler:v1.13.0
>docker tag k8s.gcr.io/kube-proxy:v1.13.0 yuyiyu/google-containers.kube-proxy:v1.13.0
>docker tag k8s.gcr.io/pause:3.1 yuyiyu/google-containers.pause:3.1
>docker tag k8s.gcr.io/etcd:3.2.24 yuyiyu/google-containers.etcd:3.2.24
>docker tag k8s.gcr.io/coredns:1.2.6 yuyiyu/google-containers.coredns:1.2.6
>
>docker push yuyiyu/google-containers.kube-apiserver:v1.13.0
docker push yuyiyu/google-containers.kube-controller-manager:v1.13.0
>docker push yuyiyu/google-containers.kube-scheduler:v1.13.0
>docker push yuyiyu/google-containers.kube-proxy:v1.13.0
>docker push yuyiyu/google-containers.pause:3.1
>docker push yuyiyu/google-containers.etcd:3.2.24
>docker push yuyiyu/google-containers.coredns:1.2.6


## 3、从国内hub.docker拉取镜像，伪装为谷歌镜像

>docker pull yuyiyu/google-containers.kube-apiserver:v1.13.0
>docker pull yuyiyu/google-containers.kube-controller-manager:v1.13.0
>docker pull yuyiyu/google-containers.kube-scheduler:v1.13.0
>docker pull yuyiyu/google-containers.kube-proxy:v1.13.0
>docker pull yuyiyu/google-containers.pause:3.1
>docker pull yuyiyu/google-containers.etcd:3.2.24
>docker pull yuyiyu/google-containers.coredns:1.2.6
>
>docker tag yuyiyu/google-containers.kube-apiserver:v1.13.0 k8s.gcr.io/kube-apiserver:v1.13.0
>docker tag yuyiyu/google-containers.kube-controller-manager:v1.13.0 k8s.gcr.io/kube-controller-manager:v1.13.0
>docker tag yuyiyu/google-containers.kube-scheduler:v1.13.0 k8s.gcr.io/kube-scheduler:v1.13.0
>docker tag yuyiyu/google-containers.kube-proxy:v1.13.0 k8s.gcr.io/kube-proxy:v1.13.0
>docker tag yuyiyu/google-containers.pause:3.1 k8s.gcr.io/pause:3.1
>docker tag yuyiyu/google-containers.etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
>docker tag yuyiyu/google-containers.coredns:1.2.6 k8s.gcr.io/coredns:1.2.6
>
>docker rmi yuyiyu/google-containers.kube-apiserver:v1.13.0
>docker rmi yuyiyu/google-containers.kube-controller-manager:v1.13.0
>docker rmi yuyiyu/google-containers.kube-scheduler:v1.13.0
>docker rmi yuyiyu/google-containers.kube-proxy:v1.13.0
>docker rmi yuyiyu/google-containers.pause:3.1
>docker rmi yuyiyu/google-containers.etcd:3.2.24
>docker rmi yuyiyu/google-containers.coredns:1.2.6

# 四、创建集群
## 1、创建master
```
kubeadm init --kubernetes-version=v1.13.0 --pod-network-cidr=10.244.0.0/16
```
>--kubernetes-version=           必须指定版本，才会使用本地镜像
>
>--pod-network-cidr=             指定pod的网段，后续 flannel 配置对应
>
>–apiserver-advertise-address    指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 2、记录下加入node的令牌

```
kubeadm join 10.27.100.165:6443 --token 8msmd9.l72acp2dlcmuoaj3 --discovery-token-ca-cert-hash sha256:2d7a4e3ebee94db507ce9e35927cdf480fb6792bdfeceede7ffc963bcbb3f9a9
```

## 3、如果忘记，在master上可查看
```
kubeadm token create --print-join-command
```

# 五、安装 flannel
## 1、下载yaml文件

```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 2、创建
```
kubectl apply -f  kube-flannel.yml
```

>clusterrole.rbac.authorization.k8s.io/flannel created
>clusterrolebinding.rbac.authorization.k8s.io/flannel created
>serviceaccount/flannel created
>configmap/kube-flannel-cfg created
>daemonset.extensions/kube-flannel-ds-amd64 created
>daemonset.extensions/kube-flannel-ds-arm64 created
>daemonset.extensions/kube-flannel-ds-arm created
>daemonset.extensions/kube-flannel-ds-ppc64le created
>daemonset.extensions/kube-flannel-ds-s390x created

## 3、修改 coreDNS 配置，指定初始DNS

```
kubectl edit cm coredns -n kube-system
```
> \#proxy . /etc/resolv.conf
> 
> proxy . 8.8.8.8

## 4、删除错误的 coreDNS 的 pod
```
kubectl get pods -n kube-system -oname |grep coredns |xargs kubectl delete -n kube-system
```

# 六、删除节点

## 1、查看节点信息

```bash
kubectl get nodes
```
>NAME                       STATUS   ROLES    AGE   VERSION
ubuntu62-virtual-machine   Ready    master   72m   v1.12.3
ubuntu63-virtual-machine   Ready    <none>   70m   v1.12.3
ubuntu64-virtual-machine   Ready    <none>   71m   v1.12.3

## 2、强制驱赶node上的pod

```bash
kubectl drain ubuntu64-virtual-machine --delete-local-data --force --ignore-daemonsets
```
>忽略warning

## 3、查看节点信息

```bash
kubectl get nodes
```

>NAME                       STATUS                     ROLES    AGE   VERSION
ubuntu62-virtual-machine   Ready                      master   73m   v1.12.3
ubuntu63-virtual-machine   Ready                      <none>   71m   v1.12.3
ubuntu64-virtual-machine   Ready,SchedulingDisabled   <none>   72m   v1.12.3

## 4、删除节点
```bash
kubectl delete node ubuntu64-virtual-machine
```

## 5、在从节点上重置集群状态
```bash
kubeadm reset
```
