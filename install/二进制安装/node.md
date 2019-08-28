# 一、将kubelet-bootstrap用户绑定到系统集群角色
```bash
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

# 二、生成proxy证书

```bash
mkdir -p /data/k8s/node/{bin,cfg,ssl}
cd /data/k8s/cfssl/api-cert
```

* vim kube-proxy-csr.json

```
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

>ca-key.pem
>
>ca.pem
>
>kube-proxy-key.pem
>
>kube-proxy.pem
>
>server-key.pem
>
>server.pem

```bash
cp ca*.pem kube*.pem server*.pem  /data/k8s/node/ssl
```

# 三、创建kubeconfig文件

```bash
cd /data/k8s/cfssl/api-cert/
```

* vim node_config.sh
>BOOTSTRAP_TOKEN的值与token.csv一致
>
>如果master高可用，需要修改KUBE_APISERVER为VIP的地址
```bash
# 创建kubelet bootstrapping kubeconfig 
BOOTSTRAP_TOKEN=d2CoHk0Q0rI3bsOICdOY1Q
KUBE_APISERVER="https://192.168.112.171:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

#----------------------

# 创建kube-proxy kubeconfig文件

kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

* 将生成的文件放入所有node中

```
cp bootstrap.kubeconfig kube-proxy.kubeconfig /data/k8s/node/cfg/
```

# 四、部署kubelet组件

## 1、将二进制包中命令复制到对应位置
```bash
cp kubelet kube-proxy /data/k8s/node/bin
```

## 2、添加kubelet的配置文件
* vim /data/k8s/node/cfg/kubelet
>修改--hostname-override为node本机地址
```bash
KUBELET_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.112.171 \
--kubeconfig=/data/k8s/node/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/data/k8s/node/cfg/bootstrap.kubeconfig \
--config=/data/k8s/node/cfg/kubelet.config \
--cert-dir=/data/k8s/node/ssl \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google-containers/pause-amd64:3.0"
```
* 配置说明
```
--hostname-override 在集群中显示的主机名
--kubeconfig 指定kubeconfig文件位置，会自动生成
--bootstrap-kubeconfig 指定刚才生成的bootstrap.kubeconfig文件
--cert-dir 颁发证书存放位置
--pod-infra-container-image 管理Pod网络的镜像
```

## 3、kubelet辅助配置文件
* vim /data/k8s/node/cfg/kubelet.config
* >修改address为node本机地址
```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1
address: 192.168.112.171
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS: ["10.0.0.2"]
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true 
```
* 参数说明
```bash
address为node的IP
clusterDNS为集群的DNS地址
```

## 4、添加kubelet为系统服务
* vim /usr/lib/systemd/system/kubelet.service

```bash
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/data/k8s/node/cfg/kubelet
ExecStart=/data/k8s/node/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable kubelet
systemctl restart kubelet
```

## 5、在Master审批Node加入集群：

>启动后还没加入到集群中，需要手动允许该节点才可以。

* 在Master节点查看请求签名的Node：

```bash
kubectl get csr
```
* 根据请求签名

```bash
kubectl certificate approve {Node的请求名称}
kubectl get node
```
* 如果设置错误的处理

```bash
# 删除对应的csr
kubectl delete csr {Node的请求名称}

# 对应节点的ssl目录删除自动生成的密钥
rm kubelet-client* kubelet.crt kubelet.key

# 重启 kubelet 然后重新签名
```

# 五、部署kube-proxy组件
## 1、添加kube-proxy的配置文件
* vim /data/k8s/node/cfg/kube-proxy
>修改--hostname-override为node本机地址
```bash
KUBE_PROXY_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=192.168.112.171 \
--cluster-cidr=10.10.0.0/16 \
--kubeconfig=/data/k8s/node/cfg/kube-proxy.kubeconfig"
```
## 2、添加kube-proxy为系统服务
* vim /usr/lib/systemd/system/kube-proxy.service
```bash
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=-/data/k8s/node/cfg/kube-proxy
ExecStart=/data/k8s/node/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl restart kube-proxy
```

# 六、查看集群状态

```bash
kubectl get node
kubectl get cs
```
