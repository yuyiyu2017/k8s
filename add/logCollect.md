# 一、收集方案

```
a、部署 DaemenSet，每个节点一个filebeat，收集其他Pod挂载到同一主机目录中的日志
   对每个Pod的日志，需要自定义日志名称，防止Pod间的输出混乱

b、每个Pod自行创建一个filebeat容器，收集自己的日志到elk系统中

c、将程序的输出都指向容器的json日志中，docker logs 和 kubectl logs，然后收集
```

# 二、filebeat示例
## 1、configMap定义filebeat配置文件
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |-
    filebeat.prospectors:
      - type: log
        paths:
          - /app-logs/www-nginx/*.log
        tags: ["nginx"]
        fields_under_root: true
        fields:
          level: info

      - type: log
        paths:
          - /app-logs/www-tomcat/catalina.out
          - /app-logs/www-tomcat/localhost_access_log*.txt
        multiline:
          pattern: '^\d+-\d+-\d+ \d+:\d+:\d+'
          negate: true
          match: after
        # exclude_lines: ['^DEBUG']
        # include_lines: ['^ERR', '^WARN']
        tags: ["tomcat"]
        fields_under_root: true
        fields:
          level: info

    output.logstash:
      hosts: ['192.168.112.171:5044']
```

## 2、filebeat与需要收集的Pod挂载同一Volume
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
    labels:
      logs: filebeat
spec:
  template:
    metadata:
      labels:
        logs: filebeat
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.1.1
        args: [
          "-c", "/usr/share/filebeat.yml",
          "-e",
        ]
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: config
          mountPath: /usr/share/filebeat.yml
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: app-logs
          mountPath: /app-logs
        - name: timezone
          mountPath: /etc/localtime
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: data
        emptyDir: {}
      - name: app-logs
        hostPath:
          path: /app-logs
      - name: timezone
        hostPath:
          path: /etc/localtime
```

# 三、Elasticsearch的helm创建
## 1、elastic官方托管的helm仓库
```
https://hub.helm.sh/charts/elastic/elasticsearch

helm repo add elastic https://helm.elastic.co

helm fetch elastic/elasticsearch --version 6.5.0
```

## 2、修改 values.yaml

```yaml
replicas: 1                     # 副本数
minimumMasterNodes: 1           # 主节点数
esJavaOpts: "-Xmx1g -Xms1g"     # JVM内存设置

resources:                      # Pod资源限制
  requests:
    cpu: "100m"
    memory: "2Gi"
  limits:
    cpu: "500m"
    memory: "2Gi"
```

## 3、创建chart
```
kubectl create namespace efk
helm install --name els1 --namespace=efk elasticsearch/
```

## 4、验证

```
kubectl run cirror-$RANDOM --rm -it --image=cirros -- /bin/sh
    创建一个小型虚拟机用于验证
    --rm    退出时即删除Pod
    -it     启用交互
```

```bash
# 解析elastic的service的IP
nslookup elasticsearch-master.efk.svc.cluster.local
    
# 查看elastic集群状态
curl elasticsearch-master.efk:9200

# 查看elastic的节点
curl elasticsearch-master.efk:9200/_cat/nodes
```

# 四、fluentd的helm创建
>收集所有Pod的控制台日志和Node的调度日志
## 1、fluentd官方托管的helm仓库
```
https://hub.helm.sh/charts/kiwigrid/fluentd-elasticsearch
helm repo add kiwigrid https://kiwigrid.github.io
helm fetch kiwigrid/fluentd-elasticsearch
```

## 2、修改 value.yaml

```yaml
elasticsearch:                      # 指向elastic的master地址
  host: 'elasticsearch-master.efk'

tolerations:                        # 如果需要监控k8s的master，需添加
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule

podAnnotations:                     # 如果启用prometheus，需添加
  prometheus.io/scrape: "true"
  prometheus.io/port: "24231"
service:
  type: ClusterIP
  ports:
    - name: "monitor-agent"
      port: 24231
```

## 3、修改镜像

```
# repository: gcr.io/google-containers/fluentd-elasticsearch
repository: yuyiyu/google-containers.fluentd-elasticsearch

docker pull gcr.io/google-containers/fluentd-elasticsearch:v2.4.0
docker tag gcr.io/google-containers/fluentd-elasticsearch:v2.4.0 yuyiyu/google-containers.fluentd-elasticsearch:v2.4.0
docker push yuyiyu/google-containers.fluentd-elasticsearch:v2.4.0
```

# 五、kibana的helm创建
## 1、kibana官方托管的helm仓库
```
https://hub.helm.sh/charts/elastic/kibana
helm repo add elastic https://helm.elastic.co
helm fetch elastic/kibana --version 6.5.0
```

## 2、修改 value.yaml

```
指向elastic的集群URL地址
    elasticsearchURL: "http://elasticsearch-master.efk.svc.cluster.local:9200"

修改镜像
    image: "yuyiyu/elastic.kibana"

docker pull docker.elastic.co/kibana/kibana:6.5.0
docker tag docker.elastic.co/kibana/kibana:6.5.0 yuyiyu/elastic.kibana:6.5.0
docker push yuyiyu/elastic.kibana:6.5.0
```
