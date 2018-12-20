kubernetes学习记录

# 国外镜像处理
## 1、国内镜像仓库
```
image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.0
修改为
image: docker.io/anjia0532/google-containers.kubernetes-dashboard-amd64:v1.10.0
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

