### k8s镜像：安装kubernetes，访问不了gcr.io怎么办？

- github开启对docker hub的读授权
- Dockerfile上github
- Docker Hub上创建Automated build
- 取到本地并push到private Registry
之前在安装k8s的时候，我们提到了依赖的gcr.io/google_containers里的镜像因为GFW的原因取不到，但是暂时没有gcr.io的国内镜像，怎么办呢？

#### 方法1：
如果有aws上的EC2虚拟机，远程到虚拟机上docker pull gcr.io/google_containers/xxx，然后tag为docker hub（也就是删掉gcr.io/google_containers/前缀），最后再docker push 到docker hub上自己的namespace下。

#### 方法2
：如果没有aws虚拟机，可以去docker hub上去捡垃圾淘宝，找到别人push上去的镜像，取下来打tag。

方法2更可行，但受制于人毕竟是不爽的，比如kubernetes 1.6.0发布了，还得去等着。有没有更好的办法呢？我从同事fenghan那里学到了一个办法。

docker hub提供了一个很棒的功能，Automated Build。

简单来说就是，你可以把你想要build的Docker Image的Dockerfile文件放到github上，然后github上开启对docker hub的授权（读权限就可以了），之后就可以在docker hub上根据这个Dockerfile来自动编译了。换句话说就是，Docker hub为我们提供了一个build的环境，你不需要本地build再push了。

这样，我可以在Github上编写一个简单的只是FROM gcr.io/google_containers/xxx的Dockerfile，然后让docker hub来build Github上的Dockerfile，他们都在墙外，build自然没有问题；然后我再通过docker hub加速器取下来，这样就可以保证是官方发布的了，速度质量都可靠。

详细说明，请参见官方文档。下面简单记录下过程。

#### github开启对docker hub的读授权
略。

#### Dockerfile上github
github上新建一个工程，比如我的Dockerfile4k8s，clone到本地。

然后在工程中新增： ./etcd-amd64/Dockerfile，Dockerfile内容为：

FROM gcr.io/google_containers/etcd-amd64:2.2.5
MAINTAINER silenceshell <silenceshell@datastart.cn>
git add/commit/push到github。

想知道google_containers里有哪些镜像，可以查询这里[https://console.cloud.google.com/gcr/images/google-containers/GLOBAL?project=google-containers]。

#### Docker Hub上创建Automated build
到docker hub上，Create -> Create Automated Build，新增一个Github类型的自动编译，选择Dockerfile4k8s项目；修改Repository的Name为etcd-amd64，简单填下描述，这样就创建了一个Automated Build。

进到Repository etcd-amd64，Build Setting中填写Dockerfile Location为etcd-amd64，修改Docker Tag Name为2.2.5，Save Change and Trigger；然后点Build Details，可以看到build的过程，状态切为Success就可以了。

#### 取到本地并push到private Registry
我们在本地用Harbor搭了一个Private Registry，所以可以给image打tag并push到自己的Registry。顺便推荐一下Harbor，很不错，vmware出品，必属精品。

docker pull silenceshell/etcd-amd64:2.2.5
docker tag silenceshell/etcd-amd64:2.2.5 {your domain}/{your namespace}/etcd-amd64:2.2.5
docker push {your domain}/{your namespace}/etcd-amd64:2.2.5
最后在k8s用的时候，别忘了tag为gcr.io/google_containers/etcd-amd64:2.2.5。

btw，如果你用的是kubeadm 1.6.0正式版本(not alpha)，那么恭喜，新版本支持自定义registry了，可以不用再自己去tag了。


#### 问题
- 每一个都需要自己写一个docker file
- 每个docker hub repository对应于一个github 相应文件
- docker hub autobuild可以根据github的tag生产相应的docker image tag(需要在规则中自己定)
- 上述方案的自动化程度不高

### 参考方案（使用了travis ci定时构建工具）
Google Container Registry(gcr.io) 中国可用镜像(长期维护)
google镜像库Google Container Registry(gcr.io) 被gfw墙了。花了点时间用github + travis ci + docker hub成功将gcr.io的全部镜像同步到docker hub了。配合 国内各种加速器 Docker 中国官方镜像加速 ,加速器 DaoCloud - 业界领先的容器云平台速度还是很快的

 使用了 travis ci 的定时构建功能，每天同步一次，同步成功后，会将结果更新到 https://github.com/anjia0532/gcr.io_mirror , 注意，同步时间为 UTC 时间，换成北京时间+8小时即可

https://anjia0532.github.io/2017/11/15/gcr-io-image-mirror/

##### travis ci定时构建未操作？？？

### 账号
- https://hub.docker.com/
- username/password: wangzg/chris3120651
- https://github.com/
- username/password: dlwangzg/chris3120651

#### 参考docker images mirror repository
gcr.mirrors.ustc.edu.cn/google_containers/kubernetes-zookeeper:1.0-3.4.10
