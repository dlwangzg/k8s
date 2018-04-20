## 安装heapster插件

### 准备YAML文件
到 [heapster release](https://github.com/kubernetes/heapster/releases) 页面 下载最新版本的 heapster
```
[centos@wangzg-k8s-4 heapster]$ wget https://github.com/kubernetes/heapster/archive/v1.5.2.zip
[centos@wangzg-k8s-4 heapster]$ unzip v1.5.2.zip
[centos@wangzg-k8s-4 heapster]$ mv v1.5.2.zip heapster-1.5.2/
```
### 修改image文件
```
deploy/kube-config/influxdb
[centos@wangzg-k8s-4 influxdb]$ cat grafana.yaml | grep image
        #image: gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
        image: harbor-ui:30814/library/heapster-grafana-amd64:v4.0.2 
```

### 执行所有定义文件
```
$ kubectl create -f deploy/kube-config/influxdb/
$ kubectl create -f deploy/kube-config/rbac/heapster-rbac.yaml
```

https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md



```
[root@wangzg-k8s-4 ~]# kubectl create configmap influxdb-config --from-file=config.toml  -n kube-system
configmap "influxdb-config" created
```
