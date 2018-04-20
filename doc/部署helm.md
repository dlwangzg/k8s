## 部署helm
Helm is a tool that streamlines installing and managing Kubernetes applications. Think of it like apt/yum/homebrew for Kubernetes.

Helm has two parts: a client (helm) and a server (tiller)
Tiller runs inside of your Kubernetes cluster, and manages releases (installations) of your charts.
Helm runs on your laptop, CI/CD, or wherever you want it to run.
Charts are Helm packages that contain at least two things:
A description of the package (Chart.yaml)
One or more templates, which contain Kubernetes manifest files
Charts can be stored on disk, or fetched from remote chart repositories (like Debian or RedHat packages)

https://docs.helm.sh/using_helm/#installing-helm
### INSTALLING THE HELM CLIENT
The Helm client can be installed either from source, or from pre-built binary releases.
#### From the Binary Releases
Every release of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.
```
[centos@wangzg-k8s-4 ~]$ wget https://kubernetes-helm.storage.googleapis.com/helm-v2.8.2-linux-amd64.tar.gz
[centos@wangzg-k8s-4 ~]$ tar -zxvf helm-v2.8.2-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/README.md
linux-amd64/LICENSE
[centos@wangzg-k8s-4 linux-amd64]$ sudo cp linux-amd64/helm /usr/local/bin/helm
```
From there, you should be able to run the client: helm help.
由于没有tiller 2.8.2的镜像，故更换client的版本为2.7.2

### INSTALLING TILLER
Tiller, the server portion of Helm, typically runs inside of your Kubernetes cluster. But for development, it can also be run locally, and configured to talk to a remote Kubernetes cluster.  
https://docs.helm.sh/using_helm/#securing-your-helm-installation  
*在本次安装中暂时不考虑helm的角色控制等问题*
```
[centos@wangzg-k8s-4 ~]$ helm init –tiller-image cnych/tiller:v2.7.2 –skip-refresh
Creating /home/centos/.helm 
Creating /home/centos/.helm/repository 
Creating /home/centos/.helm/repository/cache 
Creating /home/centos/.helm/repository/local 
Creating /home/centos/.helm/plugins 
Creating /home/centos/.helm/starters 
Creating /home/centos/.helm/cache/archive 
Creating /home/centos/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /home/centos/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!

```
我在安装过程中遇到了一些其他问题，比如初始化的时候出现了如下错误：
```
[centos@wangzg-k8s-4 ~]$ helm version
Client: &version.Version{SemVer:"v2.7.2", GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6", GitTreeState:"clean"}
E0403 03:11:04.883369   19663 portforward.go:331] an error occurred forwarding 36403 -> 44134: error forwarding port 44134 to pod 8e1008db971669e3af6b3052919ed8aab0f9fe7c7b2ebbff06daed01d6539930, uid : unable to do port forwarding: socat not found.
Error: cannot connect to Tiller

```
解决方案：在helm tiller运行的节点上安装socat可以解决
```
[centos@wangzg-k8s-4 ~]$ kubectl get pod -n kube-system -l app=helm -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP            NODE
tiller-deploy-76dcd5cc5d-zcgvw   1/1       Running   0          7m        172.30.39.3   10.0.6.8

[root@wangzg-k8s-6 kubernetes]# yum install -y socat

[centos@wangzg-k8s-4 ~]$ helm version
Client: &version.Version{SemVer:"v2.7.2", GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.7.2", GitCommit:"8478fb4fc723885b155c924d1c8c410b7a9444e6", GitTreeState:"clean"}

```

另外一个值得注意的问题是RBAC，我们的 kubernetes 集群是1.8.x版本的，默认开启了RBAC访问控制，所以我们需要为Tiller创建一个ServiceAccount，让他拥有执行的权限，详细内容可以查看 Helm 文档中的Role-based Access Control。
未创建RBAC文件提出如下信息
```
[centos@wangzg-k8s-4 helm]$ helm ls
Error: configmaps is forbidden: User "system:serviceaccount:kube-system:default" cannot list configmaps in the namespace "kube-system"

```
创建rbac.yaml文件：
```
[centos@wangzg-k8s-4 helm]$ cat rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system

[centos@wangzg-k8s-4 helm]$ kubectl create -f rbac.yaml 
serviceaccount "tiller" created
clusterrolebinding "tiller" created

[centos@wangzg-k8s-4 helm]$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
deployment "tiller-deploy" patched

[centos@wangzg-k8s-4 helm]$ helm ls
Error: could not find a ready tiller pod

```
- 创建了tiller的 ServceAccount 后还没完，因为我们的 Tiller 之前已经就部署成功了，而且是没有指定 ServiceAccount 的，所以我们需要给 Tiller 打上一个 ServiceAccount 的补丁：

### 使用
我们现在了尝试创建一个 Chart
```
$ helm create hello-helm
Creating hello-helm
$ tree hello-helm
hello-helm
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml

2 directories, 7 files
```
我们通过查看templates目录下面的deployment.yaml文件可以看出默认创建的 Chart 是一个 nginx 服务，具体的每个文件是干什么用的，我们可以前往 Helm 官方文档进行查看。
```
[centos@wangzg-k8s-4 ~]$ helm install ./hello-helm
NAME:   ignorant-bird
LAST DEPLOYED: Tue Apr  3 04:56:09 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)  AGE
ignorant-bird-hello-helm  ClusterIP  10.254.188.208  <none>       80/TCP   0s

==> v1beta1/Deployment
NAME                      DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
ignorant-bird-hello-helm  1        1        1           0          0s

==> v1/Pod(related)
NAME                                       READY  STATUS             RESTARTS  AGE
ignorant-bird-hello-helm-7b74b5678d-82tp7  0/1    ContainerCreating  0         0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=hello-helm,release=ignorant-bird" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80

```
然后我们根据提示执行下面的命令：
```
export POD_NAME=$(kubectl get pods --namespace default -l "app=hello-helm,release=kilted-bobcat" -o jsonpath="{.items[0].metadata.name}")

$ kubectl port-forward $POD_NAME 8080:80
```
- 上面操作未进？？？


参考：  
- https://blog.qikqiak.com/post/first-use-helm-on-kubernetes/
- https://blog.frognew.com/2017/12/its-time-to-use-helm.html  
- https://docs.helm.sh/using_helm/#quickstart-guide 
