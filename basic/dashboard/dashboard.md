## 1、下载yaml文件

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

## 2、修改 image 为谷歌的镜像源
```
#image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
image: docker.io/yuyiyu/google-containers.kubernetes-dashboard-amd64:v1.10.1
```

## 3、添加对外暴露端口

```yaml
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
```

## 4、语言默认与浏览器相关，强制指定为英文
```
image: docker.io/yuyiyu/google-containers.kubernetes-dashboard-amd64:v1.10.1
env:
- name: ACCEPT_LANGUAGE
  value: english
```

## 5、应用yaml文件
```
kubectl apply -f kubernetes-dashboard.yaml
```

## 6、为 dashboard 添加权限绑定
> 默认的 dashboard 用户权限比较低，绑定到集权管理角色中

* vim dashboard-admin.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

```
kubectl apply -f dashboard-admin.yaml
```

## 7、获取令牌

```
kubectl -n kube-system get secret|grep dashboard-token
```
>kubernetes-dashboard-token-9k8tx

```
kubectl -n kube-system describe secret kubernetes-dashboard-token-9k8tx
```
>Data
>>token: 令牌（可直接使用）

```
kubectl -n kube-system get secret kubernetes-dashboard-token-9k8tx -o jsonpath={.data.token}|base64 -d
```
>直接获取json将得到经过 base64 编码的 secret，需要使用 Linux 中的 base64 -d 解码
>>令牌（可直接使用）