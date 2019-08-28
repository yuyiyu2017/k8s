# kubernetes学习记录
* add
> 其他配件

* auth
> 认证体系

* basic
> 基础配件

* install
> 集群安装

* pod
> Pod相关

* service
> Service相关

* structure
> k8s简介和结构

* xls
> 表格整理

# 国外镜像处理

## 1、国内阿里云镜像仓库
```
# 如果不可用，需要自行查找可用国内镜像仓库
image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
修改为
image: registry.cn-hangzhou.aliyuncs.com/google-containers/kubernetes-dashboard-amd64:v1.10.1
```

## 2、香港自定义镜像
* 香港服务器上
```
docker pull k8s.gcr.io/pause:3.1
docker tag k8s.gcr.io/pause:3.1 yuyiyu/google-containers.pause:3.1
docker push yuyiyu/google-containers.pause:3.1
```
* 国内下载镜像
```
docker pull docker.io/yuyiyu/google-containers.pause:3.1
docker tag yuyiyu/google-containers.pause:3.1 k8s.gcr.io/pause:3.1
docker rmi yuyiyu/google-containers.pause:3.1
```

# 其他注意事项
```
centos一定要修改主机名，不能出现 yyy1.centos.com 等
否则 /etc/resolv.conf 中会出现 search centos7
导致所有的 localhost 指向了centos的服务器，而不是 127.0.0.1
K8S的DNS服务会加载宿主机的 resolv.conf，导致容器内localhost不可用
```

