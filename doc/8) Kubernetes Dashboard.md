## Kubernetes Dashboard

### Recommended setup
To access Dashboard directly (without kubectl proxy) valid certificates should be used to establish a secure HTTPS connection. They can be generated using public trusted Certificate Authorities like Let's Encrypt. Use them to replace the auto-generated certificates from Dashboard.

This setup requires, that certificates are stored in a secret named kubernetes-dashboard-certs in kube-system namespace. Assuming that you have dashboard.crt and dashboard.key files stored under $HOME/certs directory, you should create secret with contents of these files:
```
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kube-system
```
Afterwards, you are ready to deploy Dashboard using the following command:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
### Alternative setup
This setup is not fully secure. Certificates are not used and Dashboard is exposed only over HTTP. In this setup access control can be ensured only by using Authorization Header feature.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml

[centos@wangzg-k8s-4 k8s-dashboard]$ kubectl apply -f kubernetes-dashboard.yaml 
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

### 修改权限让其可以访问
#### 修改权限文件，改成
```
# ------------------- Dashboard Role & Role Binding ------------------- #
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

#### 修改service，增加NodePort
```
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort 
  ports:
  - port: 80
    targetPort: 9090
  selector:
    k8s-app: kubernetes-dashboard
```

后续要增加tls访问的设置与操作？？？？

https://github.com/kubernetes/dashboard/wiki/Access-control#authorization-header
