## 1、nginx反向代理master

```bash
stream {
    upstream k8s-apiserver {
        server 192.168.112.171:6443;
        server 192.168.112.172:6443;
    }
    server {
        # 192.168.112.175
        listen 0.0.0.0:6443;
        proxy_pass k8s-apiserver;
    }
}
```

## 2、修改node的api指向nginx后，重启

>bootstrap.kubeconfig:    
>>server: https://192.168.112.175:6443
>
>kubelet.kubeconfig:    
>>server: https://192.168.112.175:6443
>
>kube-proxy.kubeconfig:
>>server: https://192.168.112.175:6443

```
systemctl restart kube-proxy
systemctl restart kubelet
```

## 3、对nginx实行高可用

```
stream {
    upstream k8s-apiserver {
        server 192.168.112.171:6443;
        server 192.168.112.172:6443;
    }
    server {
        # 192.168.112.176
        listen 0.0.0.0:6443;
        proxy_pass k8s-apiserver;
    }
}
```

## 4、对两个nginx建立VIP
>heartbeat
## 5、修改node的api指向VIP后，重启
