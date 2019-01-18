# kubernetes学习记录
* auth
>认证体系

* basic
>基础配件

* install
>集群安装

* pod
>Pod相关

* service
>Service相关

* xls
>表格整理

# 国外镜像处理
## 1、国内阿里云镜像仓库
```
image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
修改为
image: registry.cn-hangzhou.aliyuncs.com/google-containers/kubernetes-dashboard-amd64:v1.10.0
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
docker pull yuyiyu/google-containers.pause:3.1
docker tag yuyiyu/google-containers.pause:3.1 k8s.gcr.io/pause:3.1
docker rmi yuyiyu/google-containers.pause:3.1
```

