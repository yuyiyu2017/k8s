# 一、注册网段

>Flannel要用etcd存储自身一个子网信息，在etcd中写入预定义子网段

```bash
/data/k8s/etcd/bin/etcdctl \
--ca-file=/data/k8s/etcd/ssl/ca.pem --cert-file=/data/k8s/etcd/ssl/server.pem --key-file=/data/k8s/etcd/ssl/server-key.pem \
--endpoints="https://192.168.112.171:2379,https://192.168.112.172:2379,https://192.168.112.173:2379" \
set /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```

```
"Network": "172.17.0.0/16"
    分配给docker的地址网段

"Backend": {"Type": "vxlan"}
    flannel使用vxlan模式，overlay封装网络包

"Backend": {"Type": "vxlan", "Directrouting": true}
    直接使用路由，桥接网络，二层通信

"Backend": {"Type": "host-gw"}
    Host Gateway，主机网关模式，性能更好，但宿主机不能跨网段
    各个宿主机的Pod网段不同，将宿主机的网卡当作网关
    然后按照宿主机的路由表，到达其他宿主机的网关，再到达目标Pod
```

# 二、下载安装二进制包

```bash
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
tar zxvf flannel-v0.9.1-linux-amd64.tar.gz
mkdir -p /data/k8s/flannel/{bin,cfg}
mv flanneld mk-docker-opts.sh /data/k8s/flannel/bin
```

# 三、创建flannel配置文件

* vim /data/k8s/flannel/cfg/flanneld

```bash
FLANNEL_OPTIONS="--etcd-endpoints=https://192.168.112.171:2379,https://192.168.112.172:2379,https://192.168.112.173:2379 -etcd-cafile=/data/k8s/etcd/ssl/ca.pem -etcd-certfile=/data/k8s/etcd/ssl/server.pem -etcd-keyfile=/data/k8s/etcd/ssl/server-key.pem"
```

# 四、添加systemd管理Flannel
* vim /usr/lib/systemd/system/flanneld.service
```bash
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/data/k8s/flannel/cfg/flanneld
ExecStart=/data/k8s/flannel/bin/flanneld --ip-masq $FLANNEL_OPTIONS
ExecStartPost=/data/k8s/flannel/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

# 五、配置Docker启动指定子网段
* vim /usr/lib/systemd/system/docker.service

>增加了环境变量：
>>EnvironmentFile=/run/flannel/subnet.env
>
>修改了启动选项
>>ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS

```bash
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

# 六、重启flannel和docker
```bash
systemctl daemon-reload
systemctl start flanneld
systemctl enable flanneld
systemctl restart docker
```

# 七、查看网络状态
```bash
ps -ef |grep docker
ip addr
```

>确保docker0与flannel.1在同一网段
>>测试不同节点互通，在当前节点访问另一个Node节点docker0 IP
>
>查看flannel日志方法
>>journalctl -u flannel
