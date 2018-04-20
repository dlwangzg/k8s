## 创建 CA 证书和秘钥
### 前言
kubernetes 系统各组件需要使用 TLS 证书对通信进行加密，本文档使用 CloudFlare 的 PKI 工具集 cfssl 来生成 Certificate Authority (CA) 证书和秘钥文件，CA 是自签名的证书，用来签名后续创建的其它 TLS 证书。

### 安装 CFSSL
```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ chmod +x cfssl_linux-amd64
$ sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssljson_linux-amd64
$ sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl-certinfo_linux-amd64
$ sudo mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

$ mkdir ssl
$ cd ssl
$ cfssl print-defaults config > config.json
$ cfssl print-defaults csr > csr.json

```

### 创建 CA (Certificate Authority)
#### 创建 CA 配置文件：
```
$ cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
```
字段说明
- ca-config.json：可以定义多个profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
- signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
- server auth：表示client可以用该 CA 对server提供的证书进行验证；
- client auth：表示server可以用该CA对client提供的证书进行验证；

#### 创建 CA 证书签名请求
创建 ca-csr.json 文件，内容如下：
```
$ cat ca-csr.json
{
  "CN": "kubernetes",
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
字段说明：  
- "CN"：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- "O"：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；

#### 生成 CA 证书和私钥：
```
[centos@wangzg-k8s-4 ssl]$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2018/03/28 03:11:07 [INFO] generating a new CA key and certificate from CSR
2018/03/28 03:11:07 [INFO] generate received request
2018/03/28 03:11:07 [INFO] received CSR
2018/03/28 03:11:07 [INFO] generating key: rsa-2048
2018/03/28 03:11:07 [INFO] encoded CSR
2018/03/28 03:11:07 [INFO] signed certificate with serial number 306023897612853478374033130260786180053182237529

[centos@wangzg-k8s-4 ssl]$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
### 创建 kubernetes 证书

#### 生成 kubernetes 证书和私钥
```
{
    "CN": "kubernetes",
    "hosts": [
      ""
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
- 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，由于该证书后续被 etcd 集群和 kubernetes master 集群使用，所以上面分别指定了 etcd 集群、kubernetes master 集群的主机 IP 和 kubernetes 服务的服务 IP（一般是 kube-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1）。
*我使用的hosts内容是空*
- 这是最小化安装的kubernetes集群，包括一个私有镜像仓库，三个节点的kubernetes集群，以上物理节点的IP也可以更换为主机名。
```
[centos@wangzg-k8s-4 ssl]$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
2018/03/28 03:25:28 [INFO] generate received request
2018/03/28 03:25:28 [INFO] received CSR
2018/03/28 03:25:28 [INFO] generating key: rsa-2048
2018/03/28 03:25:28 [INFO] encoded CSR
2018/03/28 03:25:28 [INFO] signed certificate with serial number 698294815885845047694725615727373737865925566117
2018/03/28 03:25:28 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

```

### 分发证书
将生成的 CA 证书、秘钥文件、配置文件拷贝到所有机器的 /etc/kubernetes/ssl 目录下
```
$ sudo mkdir -p /etc/kubernetes/ssl
$ sudo cp ca* /etc/kubernetes/ssl
```

### 校验证书
以校验 kubernetes 证书(后续部署 master 节点时生成的)为例
#### 使用 openssl 命令
```
[centos@wangzg-k8s-4 ssl]$ openssl x509  -noout -text -in  kubernetes.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7a:50:9f:5c:34:ac:10:af:b9:97:a1:e6:fc:1d:a3:0f:b5:f7:1e:a5
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Validity
            Not Before: Mar 28 03:20:00 2018 GMT
            Not After : Mar 25 03:20:00 2028 GMT
        Subject: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c3:1c:c4:0a:3f:79:bf:11:80:18:23:ad:62:29:
                    e8:fa:34:16:ea:3c:5b:64:56:ec:b7:cb:8a:3c:df:
                    36:ca:a3:91:c4:1d:0e:7e:41:b4:fd:09:5d:df:51:
                    f0:5f:7e:1b:ed:ee:40:50:88:92:af:3f:f1:39:32:
                    ed:1a:48:93:21:fe:59:5e:b8:a1:72:a6:bd:4a:b0:
                    1c:52:c7:e9:93:a5:4a:5c:24:16:f3:3d:ae:b1:8d:
                    5d:62:6d:00:55:95:90:d5:54:73:e5:1f:65:86:c4:
                    72:61:21:b2:c5:61:b9:6b:5f:de:9a:86:c4:b7:11:
                    f9:ff:85:80:91:72:6f:65:09:ae:75:37:85:ea:07:
                    45:1a:17:d4:9d:63:68:b4:df:34:ba:9b:fe:e4:73:
                    58:85:61:9c:ee:c1:d7:ff:46:13:bc:f8:48:ac:ff:
                    1d:39:fb:f4:f8:69:b4:c9:ea:89:6f:cc:57:3e:f8:
                    71:fb:95:3f:cb:24:6e:e6:68:fe:f5:ba:1d:d8:fd:
                    0b:6e:63:6b:6b:51:95:5d:5e:4d:b4:9f:11:55:45:
                    cb:2b:ed:f0:57:3d:42:e0:c8:86:c9:e5:8e:4f:4f:
                    80:4e:8f:0d:8b:52:82:06:94:fe:46:2d:b2:3e:4c:
                    70:52:04:11:6a:1d:d4:9d:08:59:e4:d8:4a:6a:c3:
                    b8:bd
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                56:75:BD:13:9A:3F:1D:7E:3C:7E:35:8E:50:92:18:F1:3A:B5:DD:93
            X509v3 Authority Key Identifier: 
                keyid:31:AE:38:A6:C1:EA:B6:01:80:2B:B7:B4:D0:DE:5E:D5:75:2D:36:69

            X509v3 Subject Alternative Name: 
                DNS:
    Signature Algorithm: sha256WithRSAEncryption
         87:4f:3a:ac:b4:0e:27:f0:66:48:b0:49:5d:79:63:23:90:6d:
         80:e6:29:3e:7f:7b:d6:0c:31:05:84:d4:77:15:fb:bc:f4:f6:
         6e:57:7d:15:72:72:cd:31:e3:dd:c2:ae:c2:ba:94:a0:ac:de:
         62:dc:47:d4:67:c1:90:ee:14:b9:03:5a:51:82:03:58:c5:2c:
         17:9b:34:18:16:5b:23:04:fb:49:53:e8:00:40:61:78:11:ba:
         06:ac:9b:0d:50:73:a8:6f:e4:ba:4f:6e:e1:3c:1b:d1:ad:9c:
         21:a9:d0:0a:ad:38:5a:fd:dc:f9:17:22:f9:f0:17:85:0c:25:
         78:82:d5:81:b6:62:c8:e2:00:68:f3:76:00:55:9e:2c:16:d6:
         b1:b7:f9:ee:48:26:da:03:94:bb:44:f4:69:49:b3:92:6d:95:
         91:64:84:92:da:fe:7a:cc:51:4b:c9:9e:b9:05:0b:85:f4:56:
         03:46:da:15:4d:10:54:37:44:e5:de:aa:a2:88:99:0b:a7:ed:
         ee:bf:cc:38:06:9e:6a:75:29:2f:16:7b:a5:03:47:a5:4a:a9:
         f3:21:ff:92:b2:1d:f4:1f:74:c0:2c:5b:47:01:bc:b3:df:1d:
         e0:38:24:c6:06:62:be:3c:88:ee:b7:33:18:bb:e2:7f:82:b0:
         47:a4:1d:e6

```
- 确认 Issuer 字段的内容和 ca-csr.json 一致；
- 确认 Subject 字段的内容和 kubernetes-csr.json 一致；
- 确认 X509v3 Subject Alternative Name 字段的内容和 kubernetes-csr.json 一致；
- 确认 X509v3 Key Usage、Extended Key Usage 字段的内容和 ca-config.json 中 kubernetes profile 一致；
