# 一、安装
## 1、安装客户端
* Ubuntu
```
snap install helm --classic
```

* Centos
```
# 必要的组键
yum install -y socat

# 二进制包安装
https://github.com/helm/helm/releases
tar -zxvf helm-v2.0.0-linux-amd64.tgz
mv linux-amd64/helm /usr/local/bin/helm
```

## 2、安装服务端Tiller
>版本必须与客户端一致
>
>服务端会以Pod形式存在，需要将其固定到主节点，防止资源紧张时被挤掉

* 如果能翻墙
```
helm init 
```
* 使用阿里仓库

```bash
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.12.1 \
    --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

## 3、验证安装和授权
* 验证
```
helm version
kubectl get pods -n kube-system
```
* 授权
>参照创建https://github.com/helm/helm/blob/master/docs/rbac.md
```bash
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

# 二、Helm
>参照 https://docs.helm.sh/using_helm/
## 1、helm结构
* 核心术语
```
chart: 一个helm程序包，包含配置清单
repository: charts仓库，http服务
release: 特定的chart部署于目标集群上的一个实例

Chart -> Config -> Release
```
* 程序架构
```
helm：客户端，管理本地的chart仓库
      与tiller服务器交互，发送chart
      实例安装、查询、卸载等操作
      
tiller：服务端，接收helm发来的charts与config，合并生成release
```

## 2、helm命令
* 常用命令示例

```bash
# 更新本地使用仓库的元数据
helm repo update

# 查看当前使用的 helm 仓库
helm repo list
    NAME  	URL                                                   
    stable	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
    local 	http://127.0.0.1:8879/charts  # 与 helm serve 命令对应

# 在helm仓库中查看稳定的chart
helm search redis

# 查看chart的详细信息
helm inspect CHART_NAME

# 安装chart
helm install --name RELEASE_NAME CHART_NAME
	程序会先fetch文件到 /root/.helm/repository/cache 再安装

# 查看当前安装的helm的release
helm list

# 删除指定的release，要完全删除 --purge
helm delete RELEASE_NAME
```

* 服务相关选项
```
completion  Generate autocompletions script for the specified shell 
home        查看当前的HELM_HOME
init        初始化helm的服务端和客户端
plugin      add, list, or remove Helm plugins
repo        add, list, remove, update, and index chart repositories
reset       uninstalls Tiller from a cluster
serve       监听8879，开启一个http服务，本地只读的helm仓库，可以发布本地chart
template    locally render templates
version     查看helm服务端和客户端的版本
```

* chart相关选项
```
create      根据名称创建chart
dependency  管理一个chart的依赖
fetch       从仓库获取chart的文件
inspect     查看chart的详细信息
install     安装chart
lint        检查chart项目的yaml格式是否正确
package     将本地的chart打包
search      从官方仓库中查找chart
verify      验证指定路径下的chart的格式是否正确可用
```

* release相关选项
```
delete      从kubernetes中删除指定的release
get         下载一个release
history     查看release历史版本
list        列出当前的release
rollback    回滚一个release到之前的版本
status      显示指定名称release的状态
test        测试一个release
upgrade     更新一个release
```

# 三、Chart
> 参照 https://docs.helm.sh/developing_charts/#charts
## 1、创建chart
* 创建一个空的chart
```
helm create myapp
```
```
wordpress/
  Chart.yaml          # 描述当前chart的元数据信息，做初始化
  LICENSE             # 可选，证书
  README.md           # 可选，此chart的说明文档
  requirements.yaml   # 可选，此chart的依赖程序或其他chart
  values.yaml         # 自定义应用到模板的数据
  charts/             # 被此chart依赖的程序打包文件
  templates/          # 模板
```

* 修改完成后语法检测

```
helm lint myapp
```

```
==> Linting myapp
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

## 2、Chart.yaml格式

```yaml
apiVersion: The chart API version, always "v1" (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
engine: gotpl # The name of the template engine (optional, defaults to gotpl)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)
tillerVersion: The version of Tiller that this chart requires. This should be expressed as a SemVer range: ">2.0.0" (optional)
```

## 3、requirements.yaml格式

```
dependencies:
  - name: apache                            # 依赖的chart名称
    version: 1.2.3                          # 依赖的chart版本
    repository: http://example.com/charts   # 依赖的chart仓库路径
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```
* helm dep up foochart
```
helm dependency update的简写
可以将依赖的chart手动下载至charts/目录中
```

## 4、templates格式
* values.yaml
```
imageRegistry: "quay.io/deis"
dockerTag: "latest"
pullPolicy: "Always"
storage: "s3"
```

* rc.yaml
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

```
{{.Values.imageRegistry}}
    代表引用 values.yaml 文件中的字段 imageRegistry

{{default "minio" .Values.storage}}
    引用 values.yaml，如果没有，则默认值为minio
```

* 安装时引用 values.yaml 
```
helm install --values=myvals.yaml wordpress
```
