## 部署 Flannel 网络
kubernetes 要求集群内各节点能通过 Pod 网段互联互通，本文档介绍使用 Flannel 在所有节点 (Master、Node) 上创建互联互通的 Pod 网段的步骤。
### 创建 TLS 秘钥和证书
etcd 集群启用了双向 TLS 认证，所以需要为 flanneld 指定与 etcd 集群通信的 CA 和秘钥。
#### 创建 flanneld 证书签名请求：
```
[centos@wangzg-k8s-4 ssl]$ cat flanneld-csr.json 
{
  "CN": "flanneld",
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

#### 生成 flanneld 证书和私钥
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

[centos@wangzg-k8s-4 ssl]$ sudo mkdir -p /etc/flanneld/ssl
[centos@wangzg-k8s-4 ssl]$ sudo cp flanneld*.pem /etc/flanneld/ssl
```

### 安装flannel网络插件
建议直接使用yum安装flanneld，除非对版本有特殊需求，默认安装的是0.7.1版本的flannel
```
[centos@wangzg-k8s-4 ssl]$ sudo yum install -y flannel
```
service配置文件/usr/lib/systemd/system/flanneld.service
```
[root@wangzg-k8s-4 ~]# cat /usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start -etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} -etcd-prefix=${FLANNEL_ETCD_PREFIX} $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```
- 修改的是ExecStart中的启动参数
```
[root@wangzg-k8s-4 ~]# cat /etc/sysconfig/flanneld 
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/flanneld/ssl/flanneld.pem -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem"

```

### 向 etcd 写入集群 Pod 网段信息
注意：本步骤只需在第一次部署 Flannel 网络时执行，后续在其它节点上部署 Flannel 时无需再写入该信息！
```
etcdctl --endpoints="https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379" --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/flanneld/ssl/flanneld.pem --key-file=/etc/flanneld/ssl/flanneld-key.pem set /kube-centos/network/config '{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```

### 启动flannel
```
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

### 现在查询etcd中的内容可以看到
```
[centos@wangzg-k8s-4 ssl]$ etcdctl --endpoints="https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379" --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/flanneld/ssl/flanneld.pem --key-file=/etc/flanneld/ssl/flanneld-key.pem ls /kube-centos/network/subnets
/kube-centos/network/subnets/172.30.51.0-24
[centos@wangzg-k8s-4 ssl]$ etcdctl --endpoints="https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379" --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/flanneld/ssl/flanneld.pem --key-file=/etc/flanneld/ssl/flanneld-key.pem get /kube-centos/network/config
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}
[centos@wangzg-k8s-4 ssl]$ etcdctl --endpoints="https://10.0.6.12:2379,https://10.0.6.9:2379,https://10.0.6.8:2379" --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/flanneld/ssl/flanneld.pem --key-file=/etc/flanneld/ssl/flanneld-key.pem ls /kube-centos/network
/kube-centos/network/config
/kube-centos/network/subnets

```

### 其它节点部署flannel
