## 部署 kubectl 命令行工具
kubectl 默认从 ~/.kube/config 配置文件获取访问 kube-apiserver 地址、证书、用户名等信息，如果没有配置该文件，执行命令时出错：
本文档介绍下载和配置 kubernetes 集群命令行工具 kubectl 的步骤。
需要将下载的 kubectl 二进制程序和生成的 ~/.kube/config 配置文件拷贝到所有使用 kubectl 命令的机器。

### 创建 admin 证书
kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和秘钥。

#### 创建 admin 证书签名请求
```
[centos@wangzg-k8s-4 ssl]$ cat admin-csr.json 
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```
- 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
- kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 所有 API的权限；
- O 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限；
- hosts 属性值为空列表；
 
#### 生成 admin 证书和私钥：
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

[centos@wangzg-k8s-4 ssl]$ ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

[centos@wangzg-k8s-4 ssl]$ sudo cp admin*.pem /etc/kubernetes/ssl/

```

### 下载 kubectl
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/1.9.6/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### 创建 kubectl kubeconfig 文件
#### 设置集群参数
```
$ kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/ssl/ca.pem --embed-certs=true --server="https://10.0.6.12:6443"
```
#### 设置客户端认证参数
```
[centos@wangzg-k8s-4 ~]$ sudo chmod 0644 /etc/kubernetes/ssl/admin-key.pem
###此步骤修改key文件的用户权限
[centos@wangzg-k8s-4 ~]$ kubectl config set-credentials admin --client-certificate=/etc/kubernetes/ssl/admin.pem --embed-certs=true --client-key=/etc/kubernetes/ssl/admin-key.pem
User "admin" set.
```

#### 设置上下文参数
```
$ kubectl config set-context kubernetes --cluster=kubernetes --user=admin
```

#### 设置默认上下文
```
$ kubectl config use-context kubernetes
```
- admin.pem 证书 O 字段值为 system:masters，kube-apiserver 预定义的 RoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 相关 API 的权限；
- 生成的 kubeconfig 被保存到 ~/.kube/config 文件；

### 分发 kubeconfig 文件
