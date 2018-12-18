# 一、生成etcd集群证书

```bash
mkdir -p /data/k8s/cfssl/etcd-cert
```

* vim ca-config.json

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
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
  "CN": "etcd CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing"
    }
  ]
}
```

* vim server-csr.json

```
{
  "CN": "etcd",
  "hosts": [
  "192.168.112.171",
  "192.168.112.172",
  "192.168.112.173"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing"
    }
  ]
}
```

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```
```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```

>ca.csr
>
>ca-key.pem
>
>ca.pem
>
>server.csr
>
>server-key.pem
>
>server.pem



# 二、安装etcd

## 1、下载二进制包

```bash
https://github.com/etcd-io/etcd
```

## 2、导入命令

```bash
mkdir /data/k8s/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.3.10-linux-amd64.tar.gz
mv etcd-v3.3.10-linux-amd64/{etcd,etcdctl} /data/k8s/etcd/bin/
```

# 三、创建etcd配置文件

* vim /data/k8s/etcd/cfg/etcd

```bash
#[Member]
ETCD_NAME="etcd02"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.112.172:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.112.172:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.112.172:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.112.172:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.112.171:2380,etcd02=https://192.168.112.172:2380,etcd03=https://192.168.112.173:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

| Item      |    Value |
| :-------- | --------:|
| ETCD_NAME | 节点名称 |
| ETCD_DATA_DIR | 数据目录|
| ETCD_LISTEN_PEER_URLS | 集群通信监听地址 |
| ETCD_LISTEN_CLIENT_URLS | 客户端访问监听地址 |
| ETCD_INITIAL_ADVERTISE_PEER_URLS | 集群通告地址 |
| ETCD_ADVERTISE_CLIENT_URLS | 客户端通告地址 |
| ETCD_INITIAL_CLUSTER | 集群节点地址 |
| ETCD_INITIAL_CLUSTER_TOKEN  | 集群Token |
| ETCD_INITIAL_CLUSTER_STATE | 加入集群的当前状态 |
>各节点ETCD_NAME和相关IP不同
>
>new是新集群，existing表示加入已有集群

# 四、添加systemd管理etcd

* vim /usr/lib/systemd/system/etcd.service

```bash
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/data/k8s/etcd/cfg/etcd
ExecStart=/data/k8s/etcd/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster=${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=new \
--cert-file=/data/k8s/etcd/ssl/server.pem \
--key-file=/data/k8s/etcd/ssl/server-key.pem \
--peer-cert-file=/data/k8s/etcd/ssl/server.pem \
--peer-key-file=/data/k8s/etcd/ssl/server-key.pem \
--trusted-ca-file=/data/k8s/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/data/k8s/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

# 五、准备和启动

## 1、将etcd证书复制到指定位置

```bash
cp ca*pem server*pem /data/k8s/etcd/ssl
```

## 2、启动并设置开启启动

```bash
systemctl start etcd
systemctl enable etcd
```

## 3、查看集群状态

```bash
/data/k8s/etcd/bin/etcdctl \
--ca-file=/data/k8s/etcd/ssl/ca.pem --cert-file=/data/k8s/etcd/ssl/server.pem --key-file=/data/k8s/etcd/ssl/server-key.pem \
--endpoints="https://192.168.112.171:2379,https://192.168.112.172:2379,https://192.168.112.173:2379" \
cluster-health
```