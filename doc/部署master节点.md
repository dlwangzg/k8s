## 部署master节点
kubernetes master 节点包含的组件：
- kube-apiserver
- kube-scheduler
- kube-controller-manager
目前这三个组件需要部署在同一台机器上。

- kube-scheduler、kube-controller-manager 和 kube-apiserver 三者的功能紧密相关；
- 同时只能有一个 kube-scheduler、kube-controller-manager 进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；

本文档介绍部署单机 kubernetes master 节点的步骤，没有实现高可用 master 集群。
计划后续再介绍部署 LB 的步骤，客户端 (kubectl、kubelet、kube-proxy) 使用 LB 的 VIP 来访问 kube-apiserver，从而实现高可用 master 集群。

master 节点与 node 节点上的 Pods 通过 Pod 网络通信，所以需要在 master 节点上部署 Flannel 网络。

### 下载最新版本的二进制文件
从 CHANGELOG页面[https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md] 下载 client 或 server tarball 文件
server 的 tarball kubernetes-server-linux-amd64.tar.gz 已经包含了 client(kubectl) 二进制文件，所以不用单独下载kubernetes-client-linux-amd64.tar.gz文件；
需要翻墙
```
[centos@wangzg-k8s-4 kubernetes]$ wget https://dl.k8s.io/v1.9.6/kubernetes-server-linux-amd64.tar.gz

[centos@wangzg-k8s-4 ~]$ tar -xzvf kubernetes-server-linux-amd64.tar.gz

[centos@wangzg-k8s-4 ~]$ cd kubernetes

[centos@wangzg-k8s-4 kubernetes]$ tar -xzvf  kubernetes-src.tar.gz

[centos@wangzg-k8s-4 ~]$ sudo cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

```

### 创建 kubernetes 证书
#### 创建 kubernetes 证书签名请求
```
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.0.6.12",
    "10.254.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```
- 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，所以上面分别指定了当前部署的 master 节点主机 IP；
- 还需要添加 kube-apiserver 注册的名为 kubernetes 的服务 IP (Service Cluster IP)，一般是 kube-apiserver --service-cluster-ip-range 选项值指定的网段的第一个IP，如 "10.254.0.1"；
$ kubectl get svc kubernetes
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.254.0.1   <none>        443/TCP   1d

#### 生成 kubernetes 证书和私钥
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

[centos@wangzg-k8s-4 ssl]$ sudo cp kubernetes*.pem /etc/kubernetes/ssl/
```

### 配置和启动 kube-apiserver
#### 创建 kube-apiserver 使用的客户端 token 文件
kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token.csv 一致，如果一致则自动为 kubelet生成证书和秘钥。

```
[centos@wangzg-k8s-4 ssl]$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
c8023ea0c8694108ce8746a2a53cf176

[centos@wangzg-k8s-4 ssl]$ cat token.csv 
c8023ea0c8694108ce8746a2a53cf176,kubelet-bootstrap,10001,"system:kubelet-bootstrap"

[centos@wangzg-k8s-4 ssl]$ sudo mv token.csv /etc/kubernetes/

```

#### 创建 kube-apiserver 的 systemd unit 文件
service配置文件/usr/lib/systemd/system/kube-apiserver.service内容：
```
[root@wangzg-k8s-4 ~]# cat /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_ETCD_SERVERS \
        $KUBE_API_ADDRESS \
        $KUBE_API_PORT \
        $KUBELET_PORT \
        $KUBE_ALLOW_PRIV \
        $KUBE_SERVICE_ADDRESSES \
        $KUBE_ADMISSION_CONTROL \
        $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

/etc/kubernetes/config文件的内容为：
```
[root@wangzg-k8s-4 ~]# cat /etc/kubernetes/config 
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://10.0.6.12:8080"
```
- --master=http://{MASTER_IP}:8080：使用非安全 8080 端口与 kube-apiserver 通信；

apiserver配置文件/etc/kubernetes/apiserver内容为：
```
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=10.0.6.12 --bind-address=10.0.6.12 --insecure-bind-address=10.0.6.12"
#
## The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"
#
## Port minions listen on
#KUBELET_PORT="--kubelet-port=10250"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota,DefaultStorageClass"
#
## Add your own!
KUBE_API_ARGS="--authorization-mode=Node,RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"
```
- 如果中途修改过--service-cluster-ip-range地址，则必须将default命名空间的kubernetes的service给删除，使用命令：kubectl delete service kubernetes，然后系统会自动用新的ip重建这个service，不然apiserver的log有报错the cluster IP x.x.x.x for service kubernetes/default is not within the service CIDR x.x.x.x/16; please recreate

- kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver通信;
- kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过安全端口访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权；
- kube-proxy、kubectl 通过在使用的证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的；
- 如果使用了 kubelet TLS Boostrap 机制，则不能再指定 --kubelet-certificate-authority、--kubelet-client-certificate 和 --kubelet-client-key 选项，否则后续 kube-apiserver 校验 kubelet 证书时出现 ”x509: certificate signed by unknown authority“ 错误；
- --admission-control 值必须包含 ServiceAccount；
- --bind-address 不能为 127.0.0.1；
- runtime-config配置为rbac.authorization.k8s.io/v1beta1，表示运行时的apiVersion；
- --service-cluster-ip-range 指定 Service Cluster IP 地址段，该地址段不能路由可达；
- 缺省情况下 kubernetes 对象保存在 etcd /registry 路径下，可以通过 --etcd-prefix 参数进行调整；
- 如果需要开通http的无认证的接口，则可以增加以下两个参数：--insecure-port=8080 --insecure-bind-address=127.0.0.1。注意，生产上不要绑定到非127.0.0.1的地址上
- 需要注意配置KUBE_API_ARGS环境变量中的--authorization-mode=Node,RBAC，增加对Node授权的模式，否则将无法注册node。
- --experimental-bootstrap-token-auth Bootstrap Token Authentication在kubernetes 1.9版本已经废弃，参数名称改为--enable-bootstrap-token-auth

#### 启动kube-apiserver
```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

### 配置和启动 kube-controller-manager
#### 创建 kube-controller-manager的serivce配置文件
文件路径/usr/lib/systemd/system/kube-controller-manager.service
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/local/bin/kube-controller-manager \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
配置文件/etc/kubernetes/controller-manager
```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-cidr=172.30.0.0/16 --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"
```
- --service-cluster-ip-range 参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致；
- --cluster-cidr 指定 Cluster 中 Pod 的 CIDR 范围，该网段在各 Node 间必须路由可达(flanneld保证)；
- --cluster-signing-* 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥；
- --root-ca-file 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件；
- --address 值必须为 127.0.0.1，kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器；
```
 $ kubectl get componentstatuses
  NAME                 STATUS      MESSAGE                                                                                        ERROR
  controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: getsockopt: connection refused
  scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: getsockopt: connection refused
```
参考：https://github.com/kubernetes-incubator/bootkube/issues/64

#### 启动 kube-controller-manager
```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

### 配置和启动 kube-scheduler
#### 创建 kube-scheduler的serivce配置文件
文件路径/usr/lib/systemd/system/kube-scheduler.service。
```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
配置文件/etc/kubernetes/scheduler
```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
```
- --address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器；

#### 启动 kube-scheduler
```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

### 验证 master 节点功能
```
[centos@wangzg-k8s-4 .kube]$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
```
