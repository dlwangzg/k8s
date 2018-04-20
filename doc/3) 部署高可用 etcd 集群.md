## 部署高可用 etcd 集群
kuberntes 系统使用 etcd 存储所有数据，本文档介绍部署一个三节点高可用 etcd 集群的步骤，这三个节点复用 kubernetes master 机器，分别命名为wangzg-k8s-4、wangzg-k8s-5、wangzg-k8s-6：
- 10.0.6.12 wangzg-k8s-4
- 10.0.6.9 wangzg-k8s-5
- 10.0.6.8 wangzg-k8s-6

本etcd集群使用的是静态配置，而非动态，更多配置参考以下链接
https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md

### 创建TLS 秘钥和证书
为了保证通信安全，客户端(如 etcdctl) 与 etcd 集群、etcd 集群之间的通信需要使用 TLS 加密，本节创建 etcd TLS 加密所需的证书和私钥。

#### 创建 etcd 证书签名请求
```
[centos@wangzg-k8s-4 ssl]$ cat etcd-csr.json 
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.0.6.12",
    "10.0.6.9",
    "10.0.6.8"
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
- hosts 字段指定授权使用该证书的 etcd 节点 IP；

#### 生成 etcd 证书和私钥
*目录/home/centos/ssl*
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

[centos@wangzg-k8s-4 ssl]$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

[centos@wangzg-k8s-4 ssl]$ sudo mkdir -p /etc/etcd/ssl
[centos@wangzg-k8s-4 ssl]$ sudo cp etcd*.pem /etc/etcd/ssl

```

#### 修改证书和私钥的所属用户为etcd用户


- 将etcd的证书与私钥、以及ca.pem文件拷贝到其他两个etcd节点

### 安装etcd
```
查看安装源中etcd的版本
$ sudo yum list etcd
etcd.x86_64  3.2.15-1.el7   extras

$ sudo yum install etcd -y

[centos@wangzg-k8s-4 ~]$ whereis etcd
etcd: /usr/bin/etcd /etc/etcd /usr/share/man/man1/etcd.1.gz

```


### 修改 etcd 的 配置 文件
主要修改ExecStart中etcd的启动参数，主要是加入ca证书，相关的参数在/etc/etcd/etcd.conf文件中进行配置
```
[centos@wangzg-k8s-4 ssl]$ cat etcd.conf 
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.0.6.12:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.0.6.12:2379,https://localhost:2379"
ETCD_NAME="wangzg-k8s-4"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.0.6.12:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.0.6.12:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
```
- 这是wangzg-k8s-4节点的配置，其他两个etcd节点只要将上面的IP地址改成相应节点的IP地址即可。ETCD_NAME换成对应节点的wangzg-k8s-4/5/6。

### 修改 etcd 的启动文件
/usr/lib/systemd/system/etcd.service
```
[root@wangzg-k8s-4 ~]# cat /usr/lib/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd --name=\"${ETCD_NAME}\" --cert-file=\"${ETCD_CERT_FILE}\" --key-file=\"${ETCD_KEY_FILE}\" --peer-cert-file=\"${ETCD_PEER_CERT_FILE}\" --peer-key-file=\"${ETCD_PEER_KEY_FILE}\" --trusted-ca-file=\"${ETCD_TRUSTED_CA_FILE}\" --peer-trusted-ca-file=\"${ETCD_PEER_TRUSTED_CA_FILE}\" --initial-advertise-peer-urls=\"${ETCD_INITIAL_ADVERTISE_PEER_URLS}\" --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" --initial-cluster=wangzg-k8s-4=https://10.0.6.12:2380,wangzg-k8s-5=https://10.0.6.9:2380,wangzg-k8s-6=https://10.0.6.8:2380 --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" --data-dir=\"${ETCD_DATA_DIR}\""
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```
- 为了保证通信安全，需要指定 etcd 的公私钥(cert-file和key-file)、Peers 通信的公私钥和 CA 证书(peer-cert-file、peer-key-file、peer-trusted-ca-file)、客户端的CA证书（trusted-ca-file）；
- --initial-cluster-state 值为 new 时，--name 的参数值必须位于 --initial-cluster 列表中；

### 启动 etcd 服务
```
[root@wangzg-k8s-5 ~]# systemctl daemon-reload
[root@wangzg-k8s-6 ~]# systemctl start etcd
[root@wangzg-k8s-6 ~]# systemctl enable etcd
```
- 在所有的 kubernetes master 节点重复上面的步骤，直到所有机器的 etcd 服务都已启动。

### 验证服务
```
[root@wangzg-k8s-4 ~]# export ETCDCTL_API=3

[root@wangzg-k8s-4 ~]# etcdctl --cacert="/etc/kubernetes/ssl/ca.pem" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" member list
INFO: 2018/03/29 08:12:29 ccBalancerWrapper: updating state and picker called by balancer: IDLE, 0xc42006dc80
INFO: 2018/03/29 08:12:29 dialing to target with scheme: ""
INFO: 2018/03/29 08:12:29 could not get resolver for scheme: ""
INFO: 2018/03/29 08:12:29 balancerWrapper: is pickfirst: false
INFO: 2018/03/29 08:12:29 balancerWrapper: got update addr from Notify: [{127.0.0.1:2379 <nil>}]
INFO: 2018/03/29 08:12:29 ccBalancerWrapper: new subconn: [{127.0.0.1:2379 0  <nil>}]
INFO: 2018/03/29 08:12:29 balancerWrapper: handle subconn state change: 0xc42022ec20, CONNECTING
INFO: 2018/03/29 08:12:29 ccBalancerWrapper: updating state and picker called by balancer: CONNECTING, 0xc42006dc80
INFO: 2018/03/29 08:12:29 balancerWrapper: handle subconn state change: 0xc42022ec20, READY
INFO: 2018/03/29 08:12:29 clientv3/balancer: pin "127.0.0.1:2379"
INFO: 2018/03/29 08:12:29 ccBalancerWrapper: updating state and picker called by balancer: READY, 0xc42006dc80
INFO: 2018/03/29 08:12:29 balancerWrapper: got update addr from Notify: [{127.0.0.1:2379 <nil>}]
2bb6069dca75652a, started, wangzg-k8s-4, https://10.0.6.12:2380, https://10.0.6.12:2379
39d226c46a05dc73, started, wangzg-k8s-6, https://10.0.6.8:2380, https://10.0.6.8:2379
902600fcc3e1a6ef, started, wangzg-k8s-5, https://10.0.6.9:2380, https://10.0.6.9:2379

[root@wangzg-k8s-4 ~]# etcdctl --cacert="/etc/kubernetes/ssl/ca.pem" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" endpoint health
127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.091585ms

[root@wangzg-k8s-4 ~]# etcdctl --endpoints=https://10.0.6.12:2379 --cacert="/etc/kubernetes/ssl/ca.pem" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" endpoint health
https://10.0.6.12:2379 is healthy: successfully committed proposal: took = 1.57715ms
[root@wangzg-k8s-4 ~]# etcdctl --endpoints=https://10.0.6.9:2379 --cacert="/etc/kubernetes/ssl/ca.pem" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" endpoint health
https://10.0.6.9:2379 is healthy: successfully committed proposal: took = 3.317444ms
[root@wangzg-k8s-4 ~]# etcdctl --endpoints=https://10.0.6.8:2379 --cacert="/etc/kubernetes/ssl/ca.pem" --cert="/etc/etcd/ssl/etcd.pem" --key="/etc/etcd/ssl/etcd-key.pem" endpoint health
https://10.0.6.8:2379 is healthy: successfully committed proposal: took = 2.79734ms
```

后续考虑将etcd的发现配置为自动发现
