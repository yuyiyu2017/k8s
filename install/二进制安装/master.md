# 一、下载master包
>选择下载二进制server包，里面会包含node和client等包
>
>kubernetes-server-linux-amd64.tar.gz
```bash
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md
```
```bash
mkdir -p /data/k8s/apiserver/{bin,cfg,ssl}
tar xzf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager kubectl /data/k8s/apiserver/bin
```

# 二、创建token文件，用于后续node节点证书签发

* vim /data/k8s/apiserver/cfg/token.csv
>d2CoHk0Q0rI3bsOICdOY1Q,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
>>第一列：随机字符串，自己可生成
>>>openssl rand -base64 16
>>>
>>>head -c 16 /dev/urandom | od -An -t x | tr -d ' '
>>
>>第二列：用户名
>>
>>第三列：UID
>>
>>第四列：用户组

# 三、api-server证书生成

```bash
mkdir -p /data/k8s/cfssl/api-cert
```

* vim ca-config.json

```
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```

* vim ca-csr.json

```
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

* vim server-csr.json
>只要使用了api端口，包括master和负载均衡器及VIP，就需要添加到hosts中

```
{
    "CN": "kubernetes",
    "hosts": [
        "10.0.0.1",
        "127.0.0.1",
        "192.168.112.171",
        "192.168.112.172",
        "192.168.112.170",
        "192.168.112.175",
        "192.168.112.176",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
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
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
```

```bash
cp ca-key.pem ca.pem server-key.pem server.pem /data/k8s/apiserver/ssl
```

* vim admin-csr.json
>用于后续 kubectl 命令管理的证书，与第八步相关
```
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

# 四、部署api-server
## 1、添加api-server的配置文件
* vim /data/k8s/apiserver/cfg/kube-apiserver 

```bash
KUBE_APISERVER_OPTS="--logtostderr=true \
--v=4 \
--etcd-servers=https://192.168.112.171:2379,https://192.168.112.172:2379,https://192.168.112.173:2379 \
--bind-address=192.168.112.171 \
--secure-port=6443 \
--advertise-address=192.168.112.171 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth \
--token-auth-file=/data/k8s/apiserver/cfg/token.csv \
--service-node-port-range=30000-50000 \
--tls-cert-file=/data/k8s/apiserver/ssl/server.pem  \
--tls-private-key-file=/data/k8s/apiserver/ssl/server-key.pem \
--client-ca-file=/data/k8s/apiserver/ssl/ca.pem \
--service-account-key-file=/data/k8s/apiserver/ssl/ca-key.pem \
--etcd-cafile=/data/k8s/etcd/ssl/ca.pem \
--etcd-certfile=/data/k8s/etcd/ssl/server.pem \
--etcd-keyfile=/data/k8s/etcd/ssl/server-key.pem"
```

* 参数说明
```bash
--logtostderr 启用日志
---v 日志等级
--etcd-servers etcd集群地址
--bind-address 监听地址
--secure-port https安全端口
--advertise-address 集群通告地址
--allow-privileged 启用授权
--service-cluster-ip-range Service虚拟IP地址段
--enable-admission-plugins 准入控制模块
--authorization-mode 认证授权，启用RBAC授权和节点自管理
--enable-bootstrap-token-auth 启用TLS bootstrap功能，后面会讲到
--token-auth-file token文件
--service-node-port-range Service Node类型默认分配端口范围
```
>各节点对应修改 --bind-address 和 --advertise-address

## 2、添加api-server为系统服务
* vim /usr/lib/systemd/system/kube-apiserver.service

```bash
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/data/k8s/apiserver/cfg/kube-apiserver
ExecStart=/data/k8s/apiserver/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl restart kube-apiserver
```

# 五、部署scheduler
## 1、添加scheduler的配置文件
* vim /data/k8s/apiserver/cfg/kube-scheduler 
```bash
KUBE_SCHEDULER_OPTS="--logtostderr=true \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect"
```
```
--master 连接本地apiserver
--leader-elect 当该组件启动多个时，自动选举（HA）
```
## 2、添加scheduler为系统服务
* vim /usr/lib/systemd/system/kube-scheduler.service 
```bash
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/data/k8s/apiserver/cfg/kube-scheduler
ExecStart=/data/k8s/apiserver/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl restart kube-scheduler
```

# 六、部署controller-manager
## 1、添加controller-manager的配置文件
* vim /data/k8s/apiserver/cfg/kube-controller-manager

```bash
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=true \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect=true \
--address=127.0.0.1 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/data/k8s/apiserver/ssl/ca.pem \
--cluster-signing-key-file=/data/k8s/apiserver/ssl/ca-key.pem  \
--root-ca-file=/data/k8s/apiserver/ssl/ca.pem \
--service-account-private-key-file=/data/k8s/apiserver/ssl/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s"
```
## 2、将controller-manager添加为系统服务
* vim /usr/lib/systemd/system/kube-controller-manager.service

```bash
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/data/k8s/apiserver/cfg/kube-controller-manager
ExecStart=/data/k8s/apiserver/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl restart kube-controller-manager
```

# 七、查看各服务状态

```bash
kubectl get cs
```
>服务均为 Healthy

# 八、如果后续要进入pod中，需创建认证
## 1、权限报错
```bash
error: unable to upgrade connection: Forbidden (user=system:anonymous, verb=create, resource=nodes, subresource=proxy)
```
## 2、应用权限

```
kubectl create -f anonymous.yaml
```

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: open-api
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: system:anonymous
```

>这个账号拥有完整权限，仅供测试使用
