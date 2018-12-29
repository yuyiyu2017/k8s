# 一、Ingress
## 1、说明
>可根据虚拟主机和其他规则，代理内网中的其他Pod
>
>对于https，只需要配置一套ssl，而不需要对每个Pod配置ssl

## 2、生产中高可用设置
>选取几台Node专门用来负载均衡
>
>在其上设置 DaemonSet
>
>然后打上特殊标签，使其他服务Pod不会调度到这些Node

## 3、yaml格式
```yaml
KIND:     Ingress
VERSION:  extensions/v1beta1

FIELDS:
    apiVersion      <string>        # 与 VERSION 对应，extensions/v1beta1
    kind            <string>        # 与 TYPE 对应，ReplicaSet
    metadata        <Object>        # 元数据
    spec            <Object>
        backend         <Object>        # 直接指定到后端地址
        rules           <[]Object>      # 通过规则映射到后端
        tls             <[]Object>      # 配置https的ssl
    status          <Object>        # 只读的状态
```

# 二、ingress安装
## 1、安装服务
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
kubectl apply -f mandatory.yaml
```

## 2、 暴露端口
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
kubectl apply -f service-nodeport.yaml
```

## 3、查看pod是否正确
```
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```

# 三、测试验证
## 1、创建nginx和apache及对应的service

```bash
kubectl run --image=nginx nginx
kubectl run --image=httpd httpd
kubectl expose deployment nginx --port=80
kubectl expose deployment httpd --port=80
```

## 2、创建ingress
>以虚拟主机（域名）区分两个service
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: http-test
spec:
  rules:
  - host: nginx.web.com
    http:
      paths:
      - backend:
          serviceName: nginx        # 与nginx的service名称对应
          servicePort: 80           # 与nginx的service端口对应
  - host: httpd.web.com
    http:
      paths:
      - backend:
          serviceName: httpd        # 与apache的service名称对应
          servicePort: 80           # 与apache的service端口对应
```

## 3、查看ingress
```
kubectl get ingresses.extensions
```

## 4、修改客户端hosts文件
```
C:\Windows\System32\drivers\etc\hosts

192.168.112.171 nginx.web.com
192.168.112.171 httpd.web.com
```
## 5、客户端访问两个域名，分别返回nginx和httpd的首页

# 四、查看配置原理

## 1、进入ingress容器中
```
kubectl exec -it -n ingress-nginx nginx-ingress-controller-76c7845cfb-w7vpz bash
```

## 2、查看nginx配置
* cat /etc/nginx/nginx.conf

```bash
upstream default-nginx-80 {
    # Load balance algorithm; empty for round robin, which is the default
    least_conn;
    keepalive 32;
    server 172.17.11.4:80 max_fails=0 fail_timeout=0;
}

upstream default-httpd-80 {
    # Load balance algorithm; empty for round robin, which is the default
    least_conn;
    keepalive 32;
    server 172.17.76.5:80 max_fails=0 fail_timeout=0;
}
```

# 五、ssl的代理

## 1、创建一对私钥和证书
* 生成私钥
```
openssl genrsa -out ssltest-key.pem 2048
```
* 用私钥签发证书
```
openssl req -new -x509 -key ssltest-key.pem -out ssltest.pem \
-subj /C=CN/ST=Beijing/O=DevOps/CN=www.ssltest.com \
-days 4000
```
```
域名：www.ssltest.com
私钥：ssltest-key.pem
证书：ssltest.pem
```
* 查看证书的信息
```
openssl x509 -in ssltest.pem -noout -text
```

## 2、将私钥和证书对应用到k8s
>tls为secret类型，ingress-secret为证书名称，创建ingress来引用
```
kubectl create secret tls ingress-secret --key ssltest-key.pem --cert ssltest.pem
```

## 3、创建ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: https-test
spec:
  tls:
  - hosts:
    - www.ssltest.com
    secretName: ingress-secret      # 指定 secret 名称
  rules:
  - host: www.ssltest.com
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```

