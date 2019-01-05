# 一、配置容器化应用的方式
## 1、自定义命令行参数
```
args: []
```

## 2、将配置文件配置在镜像中

## 3、环境变量
```
pods.spec.containers.env
```
>a、应用程序可以通过环境变量加载配置
>
>b、通过entrypoint脚本来预处理为配置文件

## 4、存储卷
>将配置文件放入对应存储卷，程序挂载后，读取卷中配置文件

# 二、环境变量例举
*  valueFrom.fieldRef 
```yaml
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

# 三、ConfigMap
## 1、yaml简要格式

```
kubectl explain configMap
```
```
KIND:     ConfigMap
VERSION:  v1

FIELDS:
   apiVersion	<string>

   binaryData	<map[string]string>     # 二进制的键值对数据

   data	<map[string]string>             # 键值对数据

   kind	<string>

   metadata	<Object>
```

## 2、用命令直接创建

```
kubectl create configmap --help
```
* --from-literal选项
>--from-literal=key1=value1 --from-literal=key2=value2
```
kubectl create configmap nginx-config --from-literal=nginx_port=80 --from-literal=server_name=myapp.yuyiyu.com
```
```
kubectl describe configmaps nginx-config
```

```
Name:         nginx-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
nginx_port:
----
80
server_name:
----
myapp.yuyiyu.com
Events:  <none>
```

* --from-file选项
>--from-file=文件路径
>>生成键为文件名，值为文件的内容

>--from-file=自定义键名=文件路径
>>生成键为自定义键名，值为文件的内容

```
kubectl create configmap nginx-file --from-file=/test/nginx_file.conf 
```

```
kubectl describe configmaps nginx-file 
```

```
Name:         nginx-file
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
nginx_file.conf:
----
{
  server_name myapp.yuyiyu.com;
  listen 80;
  root /data/web/html/;
}

Events:  <none>
```

## 3、环境变量引用configMap例举
```yaml
spec:
  containers:
  - name: myapp
    image: nginx
    env:
    - name: NGINX_SERVER_PORT
      valueFrom:
        name: nginx-config      # 指定configMap名称
        key: nginx_port         # 将configMap中的键的值赋给NGINX_SERVER_PORT
```
>环境变量引用，只有在创建Pod时才会生效，中途修改configMap不会实时更新

## 4、存储卷式引用configMap例举

```yaml
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: nginxconf2
      mountPath: /etc/nginx/config.d/
      readOnly: true
  volumes:
  - name: nginxconf2
    configMap:                  # 指定存储卷类型为 configMap
      name: nginx-file          # 指定 configMap 的名称
```

```
kubectl edit configMap nginx-file
```
>动态修改后，Pod会在一定周期内，应用最新的配置
>
>配置文件修改了，但是nginx等程序，需要重载配置，可用脚本完成

## 5、存储卷式引用部分configMap的例举

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.level: very
  special.type: charm
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: special.level
          path: keys
  restartPolicy: Never
```
>键值对 special.level 会挂载 config-volume 到 /etc/config/keys
>
>文件/etc/config/keys内容为"very"

# 四、Secret
## 1、Secret与ConfigMap区别

>ConfigMap   明文存储
>
>Secret      base64编码存储
>
>其他使用方式一样

## 2、创建方式
* 三种secret类型
```
kubectl create secret 
    docker-registry 创建一个给 Docker registry 使用的 secret
    generic         从本地 file, directory 或者 literal value 创建一个 secret
    tls             创建一个 TLS secret
```
* 创建方式与configMap类似
```
kubectl create secret generic mysql-root-password --from-literal=password=123456
```

## 3、查看Secret内容

* 查看Secret状态
```
# kubectl get secrets mysql-root-password

NAME                  TYPE     DATA   AGE
mysql-root-password   Opaque   1      9s

Opaque 代表为模糊类型
```

* 查看键值时，值不会显示

```
# kubectl describe secrets mysql-root-password

Name:         mysql-root-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
```

* 以yaml方式查看时，显示的值为编码
```
# kubectl get secrets mysql-root-password -o yaml

apiVersion: v1
data:
  password: MTIzNDU2
kind: Secret
metadata:
  creationTimestamp: "2019-01-04T08:58:21Z"
  name: mysql-root-password
  namespace: default
  resourceVersion: "478535"
  selfLink: /api/v1/namespaces/default/secrets/mysql-root-password
  uid: e3305577-0ffe-11e9-b111-000c299085a1
type: Opaque
```

* 解码后可以直接得到真实的值

```
# echo MTIzNDU2 | base64 -d

123456
```
