## 1、复制命令，并添加环境变量

## 2、设置集群项中名为kubernetes的apiserver的地址和根证书

```bash
kubectl config set-cluster kubernetes --server=https://192.168.112.171:6443 --certificate-authority=ca.pem
```

## 3、设置用户项中cluster-admin用户证书认证字段

```bash
kubectl config set-credentials cluster-admin \
--certificate-authority=/data/k8s/master/ssl/ca.pem \
--client-key=/data/k8s/master/ssl/admin-key.pem \
--client-certificate=/data/k8s/master/ssl/admin.pem
```

## 4、设置环境项中名为default的默认集群和用户

```bash
kubectl config set-context default --cluster=kubernetes --user=cluster-admin
```

## 5、设置默认环境项为default

```bash
kubectl config use-context default
```

## 6、配置文件都在 /root/.kube中

* cat config 
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/.kube/ssl/ca.pem
    server: https://192.168.112.171:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: cluster-admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: cluster-admin
  user:
    client-certificate: /root/.kube/ssl/admin.pem
    client-key: /root/.kube/ssl/admin-key.pem
```

## 7、配置admin证书
* 将master文档中创建的证书放入对应位置
>/root/.kube/ssl/
>>ca.pem
>>
>>admin.pem
>>
>>admin-key.pem

* vim admin-csr.json
>使用集群根证书签属证书
>
>"O": "system:masters" 为Group，"CN": "admin"为User
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




