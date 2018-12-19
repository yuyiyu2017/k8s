# 一、证书规划

>etcd
>>ca.pem  server.pem   server-key.pem
>
>flannel
>>ca.pem  server.pem   server-key.pem

**etcd与k8s集群证书不是一套**

>kube-apiserver
>>ca.pem  server.pem   server-key.pem
>
>kubelet
>>ca.pem  ca-key.pem
>
>kube-proxy
>>ca.pem  kube-proxy.pem  kube-proxy-key.pem
>
>kubectl
>>ca.pem  admin.pem   admin-key.pem

* cfssl工具安装
```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

# 二、服务规划

| Server      |    IP | Service|
| :-------- | --------:| --------:|
| master01 | 192.168.112.171|kube-apiserver
|||kube-controller-manager
|||kube-scheduler
|||etcd |
| master02 | 192.168.112.172|kube-apiserver
|||kube-controller-manager
|||kube-scheduler
|||etcd
| node01 | 192.168.112.171|kubelet
|||kube-proxy
|||docker
|||flannel
|||etcd
| node02 | 192.168.112.172|kubelet
|||kube-proxy
|||docker
|||flannel
| node03 | 192.168.112.173|kubelet
|||kube-proxy
|||docker
|||flannel
| Load Balancer(master) | 192.168.112.175|Nginx L4|
||192.168.112.170(VIP)||
| Load Balancer(backup) | 192.168.112.176|Nginx L4|
||192.168.112.170(VIP)||
| Registry | 192.168.112.177|Harbor|

# 三、系统软件版本
>Centos 7
>
>kubernetes 1.12
>
>Docker 18.xx-ce
>
>Etcd 3.x
>
>Flannel 0.10

# 四、etcd集群安装

# 五、安装docker

```
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm
yum install ./docker-ce-18.06.1.ce-3.el7.x86_64.rpm
```

```
systemctl start docker
systemctl enable docker
```

# 六、添加命令目录到环境变量
* vim /etc/profile
```
PATH=$PATH:/data/k8s/apiserver/bin:/data/k8s/etcd/bin:/data/k8s/node/bin
```

# 七、安装Flannel网络

# 八、部署master，apiserver、scheduler、controller-manager

# 九、部署node，kubelet、kube-proxy

# 十、master高可用方案
