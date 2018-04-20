## 部署 kubedns 插件
官方文件目录：kubernetes/cluster/addons/dns
https://github.com/kubernetes/kubernetes/tree/release-1.9/cluster/addons/dns
该插件直接使用kubernetes部署，官方的配置文件中包含以下镜像
使用的文件：
```
$ ls *.yaml *.base
kubedns-cm.yaml  kubedns-sa.yaml  kubedns-controller.yaml.base  kubedns-svc.yaml.base
```

### 修改镜像地址
```
index.tenxcloud.com/jimmy/k8s-dns-sidecar-amd64:1.14.1
```

### 系统预定义的 RoleBinding
预定义的 RoleBinding system:kube-dns 将 kube-system 命名空间的 kube-dns ServiceAccount 与 system:kube-dns Role 绑定， 该 Role 具有访问 kube-apiserver DNS 相关 API 的权限；
```
[centos@wangzg-k8s-4 kube-dns]$ kubectl get clusterrolebindings system:kube-dns -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-03-30T07:30:25Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-dns
  resourceVersion: "87"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Akube-dns
  uid: 366b73ec-33ec-11e8-bc03-fa163e603bd7
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-dns
subjects:
- kind: ServiceAccount
  name: kube-dns
  namespace: kube-system
```
- kubedns-controller.yaml 中定义的 Pods 时使用了 kubedns-sa.yaml 文件定义的 kube-dns ServiceAccount，所以具有访问 kube-apiserver DNS 相关 API 的权限。

### 执行所有定义文件
```
[centos@wangzg-k8s-4 kube-dns]$ kubectl create -f .
configmap "kube-dns" created
deployment "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created
```

### 检查 kubedns 功能
```
[centos@wangzg-k8s-4 ~]$ cat my-nginx.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

[centos@wangzg-k8s-4 ~]$  kubectl create -f my-nginx.yaml
deployment "my-nginx" created

```
Export 该 Deployment, 生成 my-nginx 服务
```
[centos@wangzg-k8s-4 ~]$ kubectl expose deploy my-nginx
service "my-nginx" exposed

[centos@wangzg-k8s-4 ~]$ kubectl get services --all-namespaces |grep my-nginx
default       my-nginx                        ClusterIP      10.254.252.142   <none>        80/TCP                       29s

```
创建另一个 Pod，查看 /etc/resolv.conf 是否包含 kubelet 配置的 --cluster-dns 和 --cluster-domain，是否能够将服务 my-nginx 解析到上面显示的 Cluster IP 10.254.86.48
```
[centos@wangzg-k8s-4 ~]$ cat pod-nginx.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80

[centos@wangzg-k8s-4 ~]$ kubectl create -f pod-nginx.yaml
pod "nginx" created

[centos@wangzg-k8s-4 kube-dns]$ kubectl exec  nginx -i -t -- /bin/bash

root@nginx:/# cat /etc/resolv.conf 
nameserver 10.254.0.2
search default.svc.cluster.local svc.cluster.local cluster.local compute.cn-dl-1.dingcloud.com
options ndots:5
root@nginx:/# ping my-nginx
PING my-nginx.default.svc.cluster.local (10.254.252.142): 48 data bytes
^C--- my-nginx.default.svc.cluster.local ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss
```
- 注意：直接ping ClusterIP是ping不通的，ClusterIP是根据IPtables路由到服务的endpoint上，只有结合ClusterIP加端口才能访问到对应的服务。
