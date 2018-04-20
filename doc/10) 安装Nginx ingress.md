## 安装Nginx ingress
Nginx ingress 使用ConfigMap来管理Nginx配置，nginx是大家熟知的代理和负载均衡软件，比起Traefik来说功能更加强大.

我们使用helm来部署，chart保存在私有的仓库中，请确保您已经安装和配置好helm，helm安装使用见使用Helm管理kubernetes应用。

### 步骤详解
#### 安装nginx-ingress chart到本地repo中
```
[centos@wangzg-k8s-4 ~]$ helm search nginx-ingress
NAME                	VERSION	DESCRIPTION                                       
stable/nginx-ingress	0.12.2 	An nginx Ingress controller that uses ConfigMap...
stable/nginx-lego   	0.3.1  	Chart for nginx-ingress-controller and kube-lego  

[centos@wangzg-k8s-4 ~]$ helm fetch stable/nginx-ingress

[centos@wangzg-k8s-4 ~]$ tar -xvf nginx-ingress-0.12.2.tgz 

```
#### 修改values.yaml配置，启用RBAC支持，
```
## Enable RBAC as per https://github.com/kubernetes/ingress/tree/master/examples/rbac/nginx and https://github.com/kubernetes/ingress/issues/266
rbac:
  #create: false
  create: true
  serviceAccountName: default

```
- 由false改为true
#### 修改values.yaml配置，替换docker image地址
```
controller:
  name: controller
  image:
    #repository: quay.io/kubernetes-ingress-controller/nginx-ingress-controller
    repository: index.tenxcloud.com/jimmy/nginx-ingress-controller
    tag: "0.9.0-beta.15"
    pullPolicy: IfNotPresent

  name: default-backend
  image:
    repository: index.tenxcloud.com/jimmy/defaultbackend
    tag: "1.3"
    pullPolicy: IfNotPresent


```

#### 打包
```
[centos@wangzg-k8s-4 nginx-ingress]$ helm package .

[centos@wangzg-k8s-4 ~]$ helm search nginx-ingress
NAME                	VERSION	DESCRIPTION                                       
local/nginx-ingress 	0.12.2 	An nginx Ingress controller that uses ConfigMap...
stable/nginx-ingress	0.12.2 	An nginx Ingress controller that uses ConfigMap...
stable/nginx-lego   	0.3.1  	Chart for nginx-ingress-controller and kube-lego 
```
#### 使用helm部署nginx-ingress
```
[centos@wangzg-k8s-4 ~]$ helm install --name nginx-ingress nginx-ingress
NAME:   nginx-ingress
LAST DEPLOYED: Wed Apr  4 01:21:12 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                      DATA  AGE
nginx-ingress-controller  1     0s

==> v1beta1/ClusterRoleBinding
NAME           AGE
nginx-ingress  0s

==> v1beta1/Role
NAME           AGE
nginx-ingress  0s

==> v1beta1/RoleBinding
NAME           AGE
nginx-ingress  0s

==> v1beta1/PodDisruptionBudget
NAME                           MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
nginx-ingress-controller       1              N/A              0                    0s
nginx-ingress-default-backend  1              N/A              0                    0s

==> v1/ServiceAccount
NAME           SECRETS  AGE
nginx-ingress  1        0s

==> v1beta1/ClusterRole
NAME           AGE
nginx-ingress  0s

==> v1/Service
NAME                           TYPE          CLUSTER-IP     EXTERNAL-IP  PORT(S)                     AGE
nginx-ingress-controller       LoadBalancer  10.254.185.44  <pending>    80:31405/TCP,443:30429/TCP  0s
nginx-ingress-default-backend  ClusterIP     10.254.71.238  <none>       80/TCP                      0s

==> v1beta1/Deployment
NAME                           DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
nginx-ingress-controller       1        1        1           0          0s
nginx-ingress-default-backend  1        1        1           0          0s

==> v1/Pod(related)
NAME                                           READY  STATUS             RESTARTS  AGE
nginx-ingress-controller-755947b477-xsp7p      0/1    ContainerCreating  0         0s
nginx-ingress-default-backend-c85879cfc-xx9zh  0/1    ContainerCreating  0         0s


NOTES:
The nginx-ingress controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace default get services -o wide -w nginx-ingress-controller'

An example Ingress that makes use of the controller:

  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

### 访问Nginx
首先获取Nginx的地址，从我们使用helm安装nginx-ingress命令的输出中那个可以看到提示，根据提示执行可以看到nginx的http和https地址：
```
[centos@wangzg-k8s-4 ~]$ kubectl --namespace default get services
NAME                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
adminserver                     ClusterIP      10.254.236.204   <none>        80/TCP                       16h
jobservice                      ClusterIP      10.254.112.159   <none>        80/TCP                       16h
kubernetes                      ClusterIP      10.254.0.1       <none>        443/TCP                      4d
mysql                           ClusterIP      10.254.68.194    <none>        3306/TCP                     16h
nginx-ingress-controller        LoadBalancer   10.254.255.99    <pending>     80:30814/TCP,443:32405/TCP   3m
nginx-ingress-default-backend   ClusterIP      10.254.139.181   <none>        80/TCP                       3m
registry                        ClusterIP      10.254.239.49    <none>        5000/TCP,5001/TCP            16h
ui                              ClusterIP      10.254.197.178   <none>        80/TCP                       16h

[centos@wangzg-k8s-4 nginx-ingress]$ kubectl --namespace default get nodes
NAME       STATUS    ROLES     AGE       VERSION
10.0.6.8   Ready     <none>    1d        v1.9.6
10.0.6.9   Ready     <none>    1d        v1.9.6

```
- 由上可知通过nodeIP+nodeport的形式访问你的应用  
 Visit http://10.0.6.8:30814 to access your application via HTTP.  
  Visit https://10.0.6.8:32405 to access your application via HTTPS.

我们分别在http和https地址上测试一下：
- /healthz返回200
- /返回404错误
```
[centos@wangzg-k8s-4 ~]$ curl -v http://10.0.6.8:30814/healthz
* About to connect() to 10.0.6.8 port 30814 (#0)
*   Trying 10.0.6.8...
* Connected to 10.0.6.8 (10.0.6.8) port 30814 (#0)
> GET /healthz HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.6.8:30814
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.13.5
< Date: Wed, 04 Apr 2018 01:58:51 GMT
< Content-Type: text/html
< Content-Length: 0
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains;
< 
* Connection #0 to host 10.0.6.8 left intact
[centos@wangzg-k8s-4 ~]$ curl -v http://10.0.6.8:30814
* About to connect() to 10.0.6.8 port 30814 (#0)
*   Trying 10.0.6.8...
* Connected to 10.0.6.8 (10.0.6.8) port 30814 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.6.8:30814
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< Server: nginx/1.13.5
< Date: Wed, 04 Apr 2018 01:59:07 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 21
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains;
< 
* Connection #0 to host 10.0.6.8 left intact
default backend - 404

[centos@wangzg-k8s-4 ~]$ curl -v --insecure https://10.0.6.8:32405/healthz
* About to connect() to 10.0.6.8 port 32405 (#0)
*   Trying 10.0.6.8...
* Connected to 10.0.6.8 (10.0.6.8) port 32405 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=Kubernetes Ingress Controller Fake Certificate,O=Acme Co
* 	start date: Apr 04 01:51:06 2018 GMT
* 	expire date: Apr 04 01:51:06 2019 GMT
* 	common name: Kubernetes Ingress Controller Fake Certificate
* 	issuer: CN=Kubernetes Ingress Controller Fake Certificate,O=Acme Co
> GET /healthz HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.6.8:32405
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.13.5
< Date: Wed, 04 Apr 2018 02:00:15 GMT
< Content-Type: text/html
< Content-Length: 0
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains;
< 
* Connection #0 to host 10.0.6.8 left intact
[centos@wangzg-k8s-4 ~]$ curl -v --insecure https://10.0.6.8:32405
* About to connect() to 10.0.6.8 port 32405 (#0)
*   Trying 10.0.6.8...
* Connected to 10.0.6.8 (10.0.6.8) port 32405 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate:
* 	subject: CN=Kubernetes Ingress Controller Fake Certificate,O=Acme Co
* 	start date: Apr 04 01:51:06 2018 GMT
* 	expire date: Apr 04 01:51:06 2019 GMT
* 	common name: Kubernetes Ingress Controller Fake Certificate
* 	issuer: CN=Kubernetes Ingress Controller Fake Certificate,O=Acme Co
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 10.0.6.8:32405
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< Server: nginx/1.13.5
< Date: Wed, 04 Apr 2018 02:00:19 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 21
< Connection: keep-alive
< Strict-Transport-Security: max-age=15724800; includeSubDomains;
< 
* Connection #0 to host 10.0.6.8 left intact
```
