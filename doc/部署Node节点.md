## 部署 Node 节点
kubernetes Node 节点包含如下组件：
- flanneld[参见部署flannel网络]
- docker[]
- kubelet
- kube-proxy

注意：每台 node 上都需要安装 flannel，master 节点上可以不安装。

### Docker安装
#### 参考Hyperledge的docker安装方法

#### Docker使用flannel网络进行配置修改
```
[root@wangzg-k8s-6 ~]# cat /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```
### 安装和配置kubelet
kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper cluster 角色(role)， 然后 kubelet 才能有权限创建认证请求(certificate signing requests)：
```
[centos@wangzg-k8s-4 .kube]$ cd /etc/kubernetes/
[centos@wangzg-k8s-4 kubernetes]$ kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
clusterrolebinding "kubelet-bootstrap" created
```
- --user=kubelet-bootstrap 是文件 /etc/kubernetes/token.csv 中指定的用户名，同时也写入了文件 /etc/kubernetes/bootstrap.kubeconfig；
- 启动 kubelet 时，如果 --kubeconfig 指定的文件不存在，则使用 bootstrap kubeconfig 向 API server 请求客户端证书。在批准 kubelet 的证书请求和回执时，将包含了生成的密钥和证书的 kubeconfig 文件写入由 -kubeconfig 指定的路径。证书和密钥文件将被放置在由 --cert-dir 指定的目录中。

### 创建 kubelet bootstrapping kubeconfig 文件
```
 # 设置集群参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server=https://10.0.6.12:6443 --kubeconfig=bootstrap.kubeconfig
Cluster "kubernetes" set.
# 设置客户端认证参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-credentials kubelet-bootstrap --token=c8023ea0c8694108ce8746a2a53cf176 --kubeconfig=bootstrap.kubeconfig
User "kubelet-bootstrap" set.
# 设置上下文参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-context default  --cluster=kubernetes --user=kubelet-bootstrap --kubeconfig=bootstrap.kubeconfig
Context "default" created.
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
- --embed-certs 为 true 时表示将 certificate-authority 证书写入到生成的 bootstrap.kubeconfig 文件中；
- 设置 kubelet 客户端认证参数时没有指定秘钥和证书，后续由 kube-apiserver 自动生成；

***以上生成的配置文件需要发分到各node节点相应目录下***
 
#### 下载最新的kubelet和kube-proxy二进制文件
```
[centos@wangzg-k8s-4 kubernetes]$ wget https://dl.k8s.io/v1.9.6/kubernetes-server-linux-amd64.tar.gz

[centos@wangzg-k8s-4 ~]$ tar -xzvf kubernetes-server-linux-amd64.tar.gz

[centos@wangzg-k8s-4 ~]$ cd kubernetes

[centos@wangzg-k8s-4 kubernetes]$ tar -xzvf  kubernetes-src.tar.gz
cp -r ./server/bin/{kube-proxy,kubelet} /usr/local/bin/
```

#### 创建kubelet的service配置文件
文件位置/usr/lib/systemd/system/kubelet.service
```
[root@wangzg-k8s-5 ~]# cat /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_POD_INFRA_CONTAINER \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
- kubelet的配置文件/etc/kubernetes/kubelet。其中的IP地址更改为你的每台node节点的IP地址。

注意：在启动kubelet之前，需要先手动创建/var/lib/kubelet目录。

下面是kubelet的配置文件/etc/kubernetes/kubelet:
```
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=10.0.6.9"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=10.0.6.9"
#
## location of the api-server
## COMMENT THIS ON KUBERNETES 1.8+
#KUBELET_API_SERVER="--api-servers=http://10.0.6.12:8080"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=index.tenxcloud.com/google_containers/pause-amd64:3.0"
#
## Add your own!
KUBELET_ARGS="--cluster-dns=10.254.0.2 --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --require-kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"
```
- 如果使用systemd方式启动，则需要额外增加两个参数--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice(未使用)
- --experimental-bootstrap-kubeconfig 在1.9版本已经变成了--bootstrap-kubeconfig 指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；
- --address 不能设置为 127.0.0.1，否则后续 Pods 访问 kubelet 的 API 接口时会失败，因为 Pods 访问的 127.0.0.1 指向自己而不是 kubelet；
如果设置了 --hostname-override 选项，则 kube-proxy 也需要设置该选项，否则会出现找不到 Node 的情况；
- "--cgroup-driver 配置成 systemd，不要使用cgroup，否则在 CentOS 系统中 kubelet 将启动失败（保持docker和kubelet中的cgroup driver配置一致即可，不一定非使用systemd）。（未配置）
- 管理员通过了 CSR 请求后，kubelet 自动在 --cert-dir 目录创建证书和私钥文件(kubelet-client.crt 和 kubelet-client.key)，然后写入 --kubeconfig 文件；
- 建议在 --kubeconfig 配置文件中指定 kube-apiserver 地址，如果未指定 --api-servers 选项，则必须指定 --require-kubeconfig 选项后才从配置文件中读取 kube-apiserver 的地址，否则 kubelet 启动后将找不到 kube-apiserver (日志中提示未找到 API Server），kubectl get nodes 不会返回对应的 Node 信息;
- 对于kuberentes1.8集群中的kubelet配置，取消了KUBELET_API_SERVER的配置，而改用kubeconfig文件来定义master地址，所以请注释掉KUBELET_API_SERVER配置
- --cluster-dns 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效；
- --cluster-domain 指定 pod 启动时 /etc/resolve.conf 文件中的 search domain ，起初我们将其配置成了 cluster.local.，这样在解析 service 的 DNS 名称时是正常的，可是在解析 headless service 中的 FQDN pod name 的时候却错误，因此我们将其修改为 cluster.local，去掉嘴后面的 ”点号“ 就可以解决该问题，关于 kubernetes 中的域名/服务名称解析请参见我的另一篇文章。
- --kubeconfig=/etc/kubernetes/kubelet.kubeconfig中指定的kubelet.kubeconfig文件在第一次启动kubelet之前并不存在，请看下文，当通过CSR请求后会自动生成kubelet.kubeconfig文件，如果你的节点上已经生成了~/.kube/config文件，你可以将该文件拷贝到该路径下，并重命名为kubelet.kubeconfig，所有node节点可以共用同一个kubelet.kubeconfig文件，这样新添加的节点就不需要再创建CSR请求就能自动添加到kubernetes集群中。同样，在任意能够访问到kubernetes集群的主机上使用kubectl --kubeconfig命令操作集群时，只要使用~/.kube/config文件就可以通过权限认证，因为这里面已经有认证信息并认为你是admin用户，对集群拥有所有权限。
- KUBELET_POD_INFRA_CONTAINER 是基础镜像容器，这里我用的是私有镜像仓库地址，大家部署的时候需要修改为自己的镜像。我上传了一个到时速云上，可以直接 docker pull index.tenxcloud.com/jimmy/pod-infrastructure 下载。pod-infrastructure镜像是Redhat制作的，大小接近80M，下载比较耗时，其实该镜像并不运行什么具体进程，可以使用Google的pause镜像gcr.io/google_containers/pause-amd64:3.0，这个镜像只有300多K，或者通过DockerHub下载jimmysong/pause-amd64:3.0。我采用的是docker pull index.tenxcloud.com/google_containers/pause-amd64
- --require-kubeconfig is deprecated. Set --kubeconfig without using --require-kubeconfig.

#### 启动 kubelet
```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

#### 通过 kubelet 的 TLS 证书请求
kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群。
查看未授权的 CSR 请求：
```
[root@wangzg-k8s-5 kubernetes]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-jFh4XzRUtyStEYroaFmJma7A9ScQXBXY_hlAy_yUYeU   32s       kubelet-bootstrap   Pending

[root@wangzg-k8s-5 kubernetes]# kubectl get nodes
No resources found.
```
通过 CSR 请求：
```
[root@wangzg-k8s-5 kubernetes]# kubectl certificate approve node-csr-jFh4XzRUtyStEYroaFmJma7A9ScQXBXY_hlAy_yUYeU
certificatesigningrequest "node-csr-jFh4XzRUtyStEYroaFmJma7A9ScQXBXY_hlAy_yUYeU" approved

[centos@wangzg-k8s-4 ~]$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
10.0.6.9   Ready     <none>    8m        v1.9.6
```

### 配置 kube-proxy
#### 创建 kube-proxy 证书
创建 kube-proxy 证书签名请求：
```
[centos@wangzg-k8s-4 ssl]$ cat kube-proxy-csr.json 
{
  "CN": "system:kube-proxy",
  "hosts": [],
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
- CN 指定该证书的 User 为 system:kube-proxy；
- kube-apiserver 预定义的 RoleBinding system:node-proxier 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；
- hosts 属性值为空列表；

#### 生成 kube-proxy 客户端证书和私钥：
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```
- 分发证书与key到相应的node节点上

#### 创建 kube-proxy kubeconfig 文件
```
# 设置集群参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true  --server=https://10.0.6.12:6443 --kubeconfig=kube-proxy.kubeconfig
Cluster "kubernetes" set.

# 设置客户端认证参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-credentials kube-proxy --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem --embed-certs=true --kubeconfig=kube-proxy.kubeconfig
User "kube-proxy" set.

# 设置上下文参数
[root@wangzg-k8s-5 kubernetes]# kubectl config set-context default --cluster=kubernetes --user=kube-proxy --kubeconfig=kube-proxy.kubeconfig
Context "default" created.

# 设置默认上下文
[root@wangzg-k8s-5 kubernetes]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
Switched to context "default".

```
- 设置集群参数和客户端认证参数时 --embed-certs 都为 true，这会将 certificate-authority、client-certificate 和 client-key 指向的证书文件内容写入到生成的 kube-proxy.kubeconfig 文件中；
- kube-proxy.pem 证书中 CN 为 system:kube-proxy，kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；
- 分发kube-proxy.kubeconfig文件到各node节点


#### 创建 kube-proxy 的service配置文件
文件路径/usr/lib/systemd/system/kube-proxy.service。
```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
kube-proxy配置文件/etc/kubernetes/proxy。
```
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=10.0.6.9 --hostname-override=10.0.6.9 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
```
#### 启动 kube-proxy
```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

### 验证集群功能
- docker 从 1.13 版本开始，可能将 iptables FORWARD chain的默认策略设置为DROP，从而导致 ping 其它 Node 上的 Pod IP 失败，遇到这种情况时，需要手动设置策略为 ACCEPT：
sudo iptables -P FORWARD ACCEPT
- 并且把以下命令写入/etc/rc.local文件中，防止节点重启iptables FORWARD chain的默认策略又还原为DROP  
```
sleep 60 && /sbin/iptables -P FORWARD ACCEPT
```

#### 关闭防火墙
```
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
$ sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
```
