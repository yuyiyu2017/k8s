# 一、其他准备

## 1、临时关闭swap
```
swapoff -a
```

## 2、永久关闭swap
```
注释 /etc/fstab
```

## 3、防火墙设置
* ubuntu处理
```bash
iptables -P FORWARD ACCEPT
```

* centos处理
```
systemctl stop firewalld
systemctl disable firewalld
systemctl stop iptables
systemctl disable iptables
```

## 4、启用ipvs模块
* vim /etc/rc.local
```bash
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe ip_vs
```

## 5、安装docker
> https://docs.docker.com/install/linux/docker-ce/centos/
> k8s 的 1.15.0 版本对应的docker版本为 18.09

## 6、修改iptables参数
```
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

## 7、修改Cgroup Driver
> https://kubernetes.io/docs/setup/production-environment/container-runtimes/
> 
> Cgroup Driver 必须为 systemd
> 
> https://k8smeetup.github.io/docs/setup/independent/troubleshooting-kubeadm/

* 查看当前的 Cgroup Driver
```
docker info | grep -i cgroup
```

* ubuntu16.04
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
```

* centos7
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload
systemctl restart docker
```

# 二、各节点安装

## 1、主机名文件
> 必须保证主机名唯一，hosts文件不需要写

* ubuntu
```tex
10.27.100.242 ubuntu62-virtual-machine
10.27.100.243 ubuntu63-virtual-machine
10.27.100.244 ubuntu64-virtual-machine
```

* centos
```tex
192.168.112.171 centos1
192.168.112.172 centos2
192.168.112.173 centos3
```

## 2、Ubuntu阿里云k8s仓库
```bash
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
```

```bash
apt-get install -y kubelet kubeadm kubectl
```

## 3、Centos阿里云k8s仓库
```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```

# 三、将谷歌镜像拉取k8s的包并tag
>必须要指定版本，这样kubeadm才不会去连k8s.io，kubeadm init同理

## 1、首先查询下当前版本需要哪些docker image

>k8s镜像的版本必须不低于 kubectl 的版本
>>kubectl version

```bash
# 如果不加 --kubernetes-version 将显示最新版本
kubeadm config images list --kubernetes-version v1.15.3
```

* 1.txt 
```
k8s.gcr.io/kube-apiserver:v1.15.3
k8s.gcr.io/kube-controller-manager:v1.15.3
k8s.gcr.io/kube-scheduler:v1.15.3
k8s.gcr.io/kube-proxy:v1.15.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

## 2、在国外服务器上拉取谷歌镜像，然后推送到国内dockerhub
* gyImage.sh 
```bash
#!/bin/bash

for i in `cat 1.txt`
do
  j=`echo "$i" | sed 's#.\+\/##g'`
  echo "$i $j"
  docker pull $i
  docker tag $i yuyiyu/google-containers.$j
  docker push yuyiyu/google-containers.$j
done
```

## 3、从国内hub.docker拉取镜像，伪装为谷歌镜像
* ygImage.sh 
```bash
#!/bin/bash

for i in `cat 1.txt`
do
  j=`echo "$i" | sed 's#.\+\/##g'`
  echo "$i $j"
  docker pull yuyiyu/google-containers.$j
  docker tag yuyiyu/google-containers.$j $i
  docker rmi yuyiyu/google-containers.$j
done
```

# 四、创建集群
## 1、创建master
```bash
kubeadm init --kubernetes-version=v1.13.0 --pod-network-cidr=10.244.0.0/16
```
>--kubernetes-version=           必须指定版本，才会使用本地镜像
>
>--pod-network-cidr=             指定pod的网段，后续 flannel 配置对应
>
>–apiserver-advertise-address    指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 2、init过程介绍

```
[init]：指定版本进行初始化操作

[preflight] ：初始化前的检查和下载所需要的Docker镜像文件

[kubelet-start] ：生成kubelet的配置文件”/var/lib/kubelet/config.yaml”，没有这个文件kubelet无法启动，所以初始化之前的kubelet实际上启动失败

[certificates]：生成Kubernetes使用的证书，存放在/etc/kubernetes/pki目录中

[kubeconfig] ：生成 KubeConfig 文件，存放在/etc/kubernetes目录中，组件之间通信需要使用对应文件

[control-plane]：使用/etc/kubernetes/manifest目录下的YAML文件，安装 Master 组件

[etcd]：使用/etc/kubernetes/manifest/etcd.yaml安装Etcd服务

[wait-control-plane]：等待control-plan部署的Master组件启动

[apiclient]：检查Master组件服务状态

[uploadconfig]：更新配置

[kubelet]：使用configMap配置kubelet

[patchnode]：更新CNI信息到Node上，通过注释的方式记录

[mark-control-plane]：为当前节点打标签，打了角色Master，和不可调度标签，这样默认就不会使用Master节点来运行Pod

[bootstrap-token]：生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到

[addons]：安装附加组件CoreDNS和kube-proxy
```


## 3、记录下加入node的令牌

```bash
kubeadm join 10.27.100.165:6443 --token 8msmd9.l72acp2dlcmuoaj3 --discovery-token-ca-cert-hash sha256:2d7a4e3ebee94db507ce9e35927cdf480fb6792bdfeceede7ffc963bcbb3f9a9
```

## 4、如果忘记，在master上可查看
```bash
kubeadm token create --print-join-command
```

## 5、加入令牌有生命周期
* 生成新令牌
```
kubeadm token generate
```

* 根据新令牌输出加入命令
```
kubeadm token create [新令牌] --print-join-command --ttl=24h
```

# 五、可以直接使用阿里云镜像
> 除非弄清所有配置，不推荐此方法，否则后续一些莫名问题

## 1、查看安装配置
```bash
kubeadm config print init-defaults > kubeadm-init.yaml
```

## 2、修改配置
```tex
imageRepository 修改成 registry.cn-hangzhou.aliyuncs.com/google_containers

serviceSubnet部分设置成 10.244.0.0/16 与 --pod-network-cidr= 参数对应

版本看情况修改

修改etcd绑定IP
```

## 3、镜像下载
```
kubeadm config images pull --config kubeadm-init.yaml
```

## 4、初始化
```
kubeadm init --config kubeadm-init.yaml
```

# 六、安装 flannel

## 1、下载yaml文件

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 2、创建
```
kubectl apply -f  kube-flannel.yml
```

## 3、coreDNS低版本报错
> https://coredns.io/plugins/loop/#troubleshooting

### 3.1、修改 coreDNS 配置，指定初始DNS
```bash
kubectl edit cm coredns -n kube-system
```

```
proxy . /etc/resolv.conf
修改为
proxy . 8.8.8.8
```

### 3.2、删除错误的 coreDNS 的 pod
```bash
kubectl get pods -n kube-system -oname |grep coredns |xargs kubectl delete -n kube-system
```

## 4、flannel报错
```
错误日志：
    Error registering network: failed to acquire lease: node "" pod cidr not assigned

原因：
    flannel的网段设置与 kubeadm 初始化时指定的网段不同
```

# 七、删除节点

## 1、查看节点信息

```bash
kubectl get nodes
```
>节点状态均为 Ready

## 2、强制驱赶node上的pod

```bash
kubectl drain ubuntu64-virtual-machine --delete-local-data --force --ignore-daemonsets
```
>忽略warning

## 3、查看节点信息

```bash
kubectl get nodes
```
>节点状态由 Ready 变更为 Ready,SchedulingDisabled  

## 4、删除节点
```bash
kubectl delete node ubuntu64-virtual-machine
```

## 5、在从节点上重置集群状态
```bash
kubeadm reset
```

# 八、增加master节点

## 1、查看当前集群的初始化配置
> kubectl -n kube-system get cm kubeadm-config -o yaml
```yaml
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta2
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns:
      type: CoreDNS
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.15.3
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
  ClusterStatus: |
    apiEndpoints:
      centos1:
        advertiseAddress: 192.168.112.171
        bindPort: 6443
    apiVersion: kubeadm.k8s.io/v1beta2
    kind: ClusterStatus
kind: ConfigMap
metadata:
  creationTimestamp: "2019-08-29T05:35:37Z"
  name: kubeadm-config
  namespace: kube-system
  resourceVersion: "167"
  selfLink: /api/v1/namespaces/kube-system/configmaps/kubeadm-config
  uid: f8bd5822-a913-4995-bc83-1446ffd149d5
```

## 2、添加集群固定出口
> 如果需要master高可用，此IP设置为VIP
```
kubectl -n kube-system edit cm kubeadm-config -o yaml
    clusterName: kubernetes
    # 添加此行
    controlPlaneEndpoint: "192.168.112.171:6443"
    controllerManager: {}
```

## 3、在首master上生成新master证书
> kubeadm init phase upload-certs --experimental-upload-certs
```
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
cb61bd061072b9b305e59a2fcd39bf3edd027b1ecc380fff3adf8babfb212132
```

## 4、新master加入集群
```
kubeadm join 192.168.112.171:6443 --token 9hspob.rjqz5p4xnzidztxy --discovery-token-ca-cert-hash \
sha256:7fedea3d28fd9c355cae256661a2e9d47565a1de9bea3f2a1c4c8099f2b91c90 \
--experimental-control-plane --certificate-key cb61bd061072b9b305e59a2fcd39bf3edd027b1ecc380fff3adf8babfb212132
```

```
在原有node加入命令后，添加参数 
--experimental-control-plane
--certificate-key cb61bd061072b9b305e59a2fcd39bf3edd027b1ecc380fff3adf8babfb212132
```

# 九、master节点的Taint

## 1、说明
> kubeadm安装的master节点是有污点的，普通服务是不会在master上创建pod

* 查看节点的taints
```
kubectl describe node centos3

Taints: node-role.kubernetes.io/master:NoSchedule
```

## 2、为节点设置污点
> 二进制安装的节点没有设置taints，可以添加
```
kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule
```

## 3、取消节点的污点
```
# 在键值后加上'减号'即可
kubectl taint node centos3 node-role.kubernetes.io/master:NoSchedule-
```

# 十、master节点的Role

## 1、查看节点的Role

* kubectl get nodes
```
NAME      STATUS   ROLES    AGE    VERSION
centos1   Ready    master   111m   v1.15.3
centos2   Ready    <none>   109m   v1.15.3
centos3   Ready    master   49m    v1.15.3
```

* kubectl describe nodes centos3
```
Name:               centos3
Roles:              master
Labels:             node-role.kubernetes.io/master=
```

## 2、通过标签控制Role
```
# 设置为master
kubectl label node centos3 node-role.kubernetes.io/master=

# 取消master
kubectl label node centos3 node-role.kubernetes.io/master-
```


