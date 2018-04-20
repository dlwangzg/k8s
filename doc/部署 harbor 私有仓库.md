## 部署 harbor 私有仓库
本文档介绍使用 k8s 部署 harbor 私有仓库的步骤，你也可以使用 docker 官方的 registry 镜像部署私有仓库(部署 Docker Registry)。
#### 下载
当前版本是 1.2.2
```
[centos@wangzg-k8s-4 ~]$ wget http://harbor.orientsoft.cn/harbor-1.2.2/harbor-offline-installer-v1.2.2.tgz
```
#### 准备 Docker 镜像
解压以后要把所有镜像上传到 k8s 工作节点。
```
[centos@wangzg-k8s-4 ~]$ tar -zvxf harbor-offline-installer-v1.2.2.tgz 
[centos@wangzg-k8s-4 ~]$ cd harbor
[centos@wangzg-k8s-4 harbor]$ scp -i /home/centos/cloud-key harbor.v1.2.2.tar.gz centos@10.0.6.9:~

[root@wangzg-k8s-5 ~]# docker load -i /home/centos/harbor.v1.2.2.tar.gz 
[root@wangzg-k8s-5 ~]# docker images
vmware/harbor-log                                   v1.2.2              36ef78ae27df        5 months ago        200MB
vmware/harbor-jobservice                            v1.2.2              e2af366cba44        5 months ago        164MB
vmware/harbor-ui                                    v1.2.2              39efb472c253        5 months ago        178MB
vmware/harbor-adminserver                           v1.2.2              c75963ec543f        5 months ago        142MB
vmware/harbor-db                                    v1.2.2              ee7b9fa37c5d        5 months ago        329MB
vmware/nginx-photon                                 1.11.13             6cc5c831fc7f        5 months ago        144MB
vmware/registry                                     2.6.2-photon        5d9100e4350e        7 months ago        173MB
vmware/postgresql                                   9.6.4-photon        c562762cbd12        7 months ago        225MB
vmware/clair                                        v2.0.1-photon       f04966b4af6c        8 months ago        297MB
vmware/harbor-notary-db                             mariadb-10.1.10     64ed814665c6        12 months ago       324MB
vmware/notary-photon                                signer-0.5.0        b1eda7d10640        12 months ago       156MB
vmware/notary-photon                                server-0.5.0        6e2646682e3c        12 months ago       157MB

```

#### 准备配置文件
下载源码
```
[centos@wangzg-k8s-4 harbor-code]$ git clone https://github.com/vmware/harbor.git
```
在以下目录中所有的 rc.yaml 中镜像替换成正确的镜像地址
```
make/kubernetes/**/*.deploy.yaml

如：
[centos@wangzg-k8s-4 adminserver]$ cat adminserver.deploy.yaml | grep image
        image: vmware/harbor-adminserver:v1.2.2
        imagePullPolicy: IfNotPresent
```

在以下目录文件中设置存储的容量
```
make/kubernetes/pv/*.pvc.yaml: Persistent Volume Claim。你可以设置存储这些文件的容量，例如：
[centos@wangzg-k8s-4 pv]$ cat registry.pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 60Gi
  selector:
    matchLabels:
      type: registry

```
如果你改变了 PVC 的容量，那么你也需要相应的设置 PV 的容量。
如果想让外部访问，需要修改两个地方
```
/home/centos/harbor-code/harbor/make/kubernetes/pv
[centos@wangzg-k8s-4 pv]$ ll
total 24
-rw-rw-r-- 1 centos centos 201 Apr  3 05:59 log.pvc.yaml
-rw-rw-r-- 1 centos centos 230 Apr  3 05:59 log.pv.yaml
-rw-rw-r-- 1 centos centos 212 Apr  3 06:15 registry.pvc.yaml
-rw-rw-r-- 1 centos centos 245 Apr  3 06:15 registry.pv.yaml
-rw-rw-r-- 1 centos centos 209 Apr  3 05:59 storage.pvc.yaml
-rw-rw-r-- 1 centos centos 241 Apr  3 05:59 storage.pv.yaml

[centos@wangzg-k8s-4 pv]$ cat registry.pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
  labels:
    type: registry
spec:
  capacity:
    storage: 60Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/registry

```
将上述相关的参数修改完成后，执行下面的命令生成ConfigMap文件 
```
python make/kubernetes/k8s-prepare

脚本执行完成后会生成下面的一些文件：
- make/kubernetes/jobservice/jobservice.cm.yaml
- make/kubernetes/mysql/mysql.cm.yaml
- make/kubernetes/registry/registry.cm.yaml
- make/kubernetes/ui/ui.cm.yaml
```

### 运行
```
# create pv & pvc
$ kubectl apply -f make/kubernetes/pv/ops.pv.yaml

# create config map
$ kubectl apply -f make/kubernetes/adminserver/adminserver.cm.yaml
$ kubectl apply -f make/kubernetes/jobservice/jobservice.cm.yaml
$ kubectl apply -f make/kubernetes/mysql/mysql.cm.yaml
$ kubectl apply -f make/kubernetes/registry/registry.cm.yaml
$ kubectl apply -f make/kubernetes/ui/ui.cm.yaml

# create service
$ kubectl apply -f make/kubernetes/adminserver/adminserver.svc.yaml
$ kubectl apply -f make/kubernetes/jobservice/jobservice.svc.yaml
$ kubectl apply -f make/kubernetes/mysql/mysql.svc.yaml
$ kubectl apply -f make/kubernetes/registry/registry.svc.yaml
$ kubectl apply -f make/kubernetes/ui/ui.svc.yaml

# create k8s deployment/statefulset
$ kubectl apply -f make/kubernetes/adminserver/adminserver.rc.yaml
$ kubectl apply -f make/kubernetes/registry/registry.rc.yaml
$ kubectl apply -f make/kubernetes/mysql/mysql.rc.yaml
$ kubectl apply -f make/kubernetes/jobservice/jobservice.rc.yaml
$ kubectl apply -f make/kubernetes/ui/ui.rc.yaml
```
上面的相关yaml 文件执行完成后，我们就可以通过nginx ingress给上面的ui绑定一个域名:
```
[centos@wangzg-k8s-4 kubernetes]$ cat ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: harbor
spec:
  rules:
  - host: harbor-ui 
    http:
      paths:
      - path: /
        backend:
          serviceName: ui
          servicePort: 80
      - path: /v2
        backend:
          serviceName: registry
          servicePort: repo
      - path: /service
        backend:
          serviceName: ui
          servicePort: 80

[centos@wangzg-k8s-4 kubernetes]$ kubectl create -f ingress.yaml 
```

### 访问
#### 浏览器访问
根据nginx ingress开放端口进行访问
本示例中nginx ingress node port的端口为30814(172.17.20.31:30814)
故需在所要访问的机器上配置hosts文件记录内容如下：
```
172.17.20.31 harbor-ui
```
在浏览器直接访问
http://harbor-ui:30814

默认username/password: admin/Harbor12345

#### docker login
```
在/etc/docker/daemon.json中添加insecure-registry：

{
    "insecure-registries": ["harbor-ui:30814"]
}

重启docker service生效
[root@wangzg-k8s-4 ~]# docker login http://harbor-ui:30814
Username: wangzg
Password: 

[root@wangzg-k8s-4 ~]# docker tag index.tenxcloud.com/jimmy/heapster-grafana-amd64:v4.0.2 harbor-ui:30814/library/heapster-grafana-amd64:v4.0.2
[root@wangzg-k8s-4 ~]# docker push harbor-ui:30814/library/heapster-grafana-amd64:v4.0.2
```

##### troubleshoot
```
[root@wangzg-k8s-4 ~]# docker login http://harbor-ui:30814
Username: wangzg
Password: 
Error response from daemon: Get https://harbor-ui:30814/v2/: http: server gave HTTP response to HTTPS client

解决方法
在/etc/docker/daemon.json中添加insecure-registry：

{
    "insecure-registries": ["hub.tonybai.com:8070"]
}

重启docker service生效
```
### 上传镜像到私有库中
```
[root@wangzg-k8s-6 kubernetes]# docker push harbor-ui:30814/library/tiller:v2.7.2
The push refers to repository [harbor-ui:30814/library/tiller]
c67d1b51815b: Pushing [==================================================>]  50.63MB/50.63MB
9cefb5c424ea: Layer already exists 
7e2d3752fd4f: Pushing [==================================================>]  4.809MB/4.809MB
error parsing HTTP 413 response body: invalid character '<' looking for beginning of value: "<html>\r\n<head><title>413 Request Entity Too Large</title></head>\r\n<body bgcolor=\"white\">\r\n<center><h1>413 Request Entity Too Large</h1></center>\r\n<hr><center>nginx/1.13.5</center>\r\n</body>\r\n</html>\r\n"

```

解决方法(在dashboard的配置集页面上修改即可)
```
{
  "kind": "ConfigMap",
  "apiVersion": "v1",
  "metadata": {
    "name": "nginx-ingress-controller",
    "namespace": "default",
    "selfLink": "/api/v1/namespaces/default/configmaps/nginx-ingress-controller",
    "uid": "a20e29a2-37aa-11e8-bc03-fa163e603bd7",
    "resourceVersion": "919910",
    "creationTimestamp": "2018-04-04T01:51:03Z",
    "labels": {
      "app": "nginx-ingress",
      "chart": "nginx-ingress-0.12.2",
      "component": "controller",
      "heritage": "Tiller",
      "release": "nginx-ingress"
    }
  },
  "data": {
    "enable-vts-status": "false",
    "proxy-body-size": "1000m"
  }
}
```

```
[root@wangzg-k8s-6 kubernetes]# docker push harbor-ui:30814/library/tiller
The push refers to repository [harbor-ui:30814/library/tiller]
c67d1b51815b: Pushed 
9cefb5c424ea: Layer already exists 
7e2d3752fd4f: Pushed 
v2.7.2: digest: sha256:91b8ea8b2f46b2e39341c44b26938333d76bade1686dfaea9f52e46d70650636 size: 950
```

#### 在Kubernetes集群上部署高可用Harbor镜像仓库
此部分未进行操作，？？？？  
https://tonybai.com/2017/12/08/deploy-high-availability-harbor-on-kubernetes-cluster/

#### Helm Chart for Harbor
这部分现在做不了，因为helm的版本是2.7.2，最新2.8.2版本的tiller docker image没有
https://github.com/vmware/harbor/tree/master/contrib/helm/harbor  
留待后续进行操作？？？？

#### 问题（待解决）
通过docker login上传的镜像在harbor ui上看不到
猜测有可能是harbor-ui:30814/library/tiller，加上了端口号的原因
