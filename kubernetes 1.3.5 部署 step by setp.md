kubernetes（简称k8s）是google开源的容器集群管理系统。使用kubernetes，可以方便的实现对容器的调度服务、负载均衡、自动伸缩等。本文主要介绍如何在centos7.2上部署kubernetes1.3.5集群。

<!-- more -->

# k8s集群相关组件

## master节点
* etcd：etcd是一个高可用的键值存储系统，主要用于共享配置和服务发现。k8s使用etcd存储集群信息。
* kube-apiserver：封装etcd操作，实现了一套REST接口。k8s其他组件通过apiserver进行交互。
* kube-controller-manager：简称KCM，通过apiserver监听资源（容器、node）的变化并执行相应操作。
* kube-scheduler：通过apiserver监听Pod的变化，对Pod进行调度。
* kubectl：k8s提供的一个CLI程序，可以通过kebectl操作apiserver

## node节点
* flannel：CoreOS 团队针对 Kubernetes 设计的一个覆盖网络（Overlay Network）工具。可以实现每个node上的所有容器都在同一个子网中，并且不同node上的容器可以直接通信。
* kubelet：相当于kubernetes在每个node节点上的agent。负责创建、监控、删除容器以及监控node。
* kube-proxy：service代理及负载均衡
* docker：
* bridge-util：linux网桥管理套件

# 准备环境

安装新版的docker：
```shell
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
 
yum install docker-engine -y
```

设置hosts，修改`/etc/hosts`：
```
127.0.0.1  localhost  localhost.localdomain  VM_82_192_centos
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
<youre master ip here> kube-master
<your node1 ip here> kube-node1
<your node2 ip here> kube-node2
```

下载k8s源码，编辑`kubernetes\cluster\centos\build.sh`，修改要下载的k8s版本：

```shell
...
# Define flannel version to use.
FLANNEL_VERSION=${FLANNEL_VERSION:-"0.5.5"}
 
# Define etcd version to use.
ETCD_VERSION=${ETCD_VERSION:-"2.3.7"}
 
# Define k8s version to use.
K8S_VERSION=${K8S_VERSION:-"1.3.5"}
...
```
运行`./build.sh all`下载并提前对应版本的k8s到binaries目录中。



```log
[root@VM_82_192_centos centos]# ./build.sh all
Download flannel release v0.5.5 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   608    0   608    0     0    474      0 --:--:--  0:00:01 --:--:--   475
100 3408k  100 3408k    0     0   647k      0  0:00:05  0:00:05 --:--:-- 1010k
Download etcd release v2.3.7 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   606    0   606    0     0    543      0 --:--:--  0:00:01 --:--:--   543
100 8347k  100 8347k    0     0  1430k      0  0:00:05  0:00:05 --:--:-- 1981k
Download kubernetes release v1.3.5 ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   593    0   593    0     0    549      0 --:--:--  0:00:01 --:--:--   549
100 1418M  100 1418M    0     0  5921k      0  0:04:05  0:04:05 --:--:-- 6571k
Download docker-latest ...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 76963  100 76963    0     0  73290      0  0:00:01  0:00:01 --:--:-- 73367
/tmp/downloads/kubernetes/server ~/gopath/centos
~/gopath/centos
Done! All binaries are stored in /root/gopath/centos/binaries
```


> 注：如果想要自己编译k8s的话，主机的内存应在4G，硬盘应在10G以上。或者通过以下命令添加一个swap：
```shell
dd if=/dev/zero of=/var/myswap bs=1M count=4096
mkswap /var/myswap
swapon /var/myswap
swapon -s
```


这里为了方便起见，我使用了nfs在多个主机间共享文件，关于nfs的设置，可以参考：[Configure NFS Server](https://www.server-world.info/en/note?os=CentOS_7&p=nfs)、[CentOS 7 NFS服务器和客户端设置](http://blog.huatai.me/2014/10/14/CentOS-7-NFS-Server-and-Client-Setup/)

接下来，我们将使用这些二进制包搭建一个k8s集群。

# 配置ETCD

修改`/etc/etcd/etcd.conf`中的localhost为kube-master
执行：
```shell
systemctl start etcd
systemctl enable etcd
alias etcdctl='etcdctl --peers="http://kube-master:2379"'
etcdctl ls
```

# 启动apiserver
```shell
./kube-apiserver --logtostderr=false --v=0 --etcd-servers=http://kube-master:2379 --insecure-bind-address=0.0.0.0 --allow-privileged=false --service-cluster-ip-range=10.254.0.0/16 --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota --log-dir=/data/kubernetes --secure-port=0
```

# 启动KCM
```shell
./kube-controller-manager
```

# 启动scheduler
```shell
./kube-scheduler
```

使用kubectl验证apiserver是否正常启动：
```shell
# ./kubectl  version
Client Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.5", GitCommit:"b0deb2eb8f4037421077f77cb163dbb4c0a2a9f5", GitTreeState:"clean", BuildDate:"2016-08-11T20:29:08Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.5", GitCommit:"b0deb2eb8f4037421077f77cb163dbb4c0a2a9f5", GitTreeState:"clean", BuildDate:"2016-08-11T20:21:58Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
```

# 配置flannel
在etcd中写入flannel配置，每个node节点上的flannel将会从这个大的CIDR中随机分配一个不冲突的子网给docker使用：
```shell
etcdctl   mk /kube/network/config '{"Network":"10.254.0.0/16"}
```

在node节点上编辑`/etc/sysconfig/flanneld`：
```txt
FLANNEL_ETCD="http://kube-master:2379"
FLANNEL_ETCD_KEY="/kube/network"
```

在node上启动flannel：

```shell
systemctl start flannel
systemctl enable flannel
systemctl start docker
```

使用ifconfig查看docker0网桥网关是否在同一个网段中，如果不一致需要执行以下命令修改：

```shell
systemctl stop docker
ifconfig docker0 down
brctl delbr docker0
systemctl start docker
```

# 在node上启动kubelet

```shell
./kubelet --logtostderr=true --v=0 --api-servers=http://kube-master:8080 --address=0.0.0.0 --port=10250 --hostname-override=kube-node1 --allow-privileged=false --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest --log-dir=/data/kubernetes
```

# 在node上启动kube-proxy

```shell
./kube-proxy --logtostderr=true --v=0 --master=http://kube-master:8080
```

# 测试集群是否可以正常工作

在master节点上使用kubectl创建一个redis容器：
```shell
kubectl run redis --image=redis
```

# 配置Dashboard
进入`/kubernetes/cluster/addons/dashboard`目录，注意由于这里的配置默认把RC和Service都创建在了kube-system这个namespace下，导致不能直接使用kubectl命令查看相关信息，所以我这里将`namespace: kube-system`给注释掉了，关于namespace的更多信息，可以参考：[Namespaces](http://kubernetes.io/docs/user-guide/namespaces/)。同时需要给dashboard添加一个参数，以便dashboard能够正确访问apiserver：
```yaml
    spec:
      containers:
      - name: kibernetes-dashboard
        image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.1.1
        imagePullPolicy: IfNotPresent // 如果node节点上已经有这个镜像了，就不再拉取。默认每次创建时都会拉取镜像
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        ports:
        - containerPort: 9090
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
        args:
        - --apiserver-host=http:/ / <your master ip here> :8080 // 这里要改为master节点的ip
```

修改dashboard-service.yaml，设置Node Port，这样访问节点的ip+port就可以访问dashboard了：
```yaml
# This file should be kept in sync with cluster/gce/coreos/kube-manifests/addons/dashboard/dashboard-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-dashboard
  labels:
    k8s-app: k8s-dashboard
    #kubernetes.io/cluster-service: "true"
spec:
  type: NodePort
  selector:
    k8s-app: k8s-dashboard
  ports:
  - port: 80
    targetPort: 9090
    nodePort: 30001
```

创建dashboard rc和service:
```shell
# kubectl create -f dashboard-controller.yaml
# kubectl create -f dashboard-service.yaml
```

使用kubectl查看Pod的信息：

```shell
# kubectl get pod
NAME                         READY     STATUS    RESTARTS   AGE
k8s-dashboard-v1.1.1-4h4df   1/1       Running   0          55m
k8s-dashboard-v1.1.1-x6sl6   1/1       Running   0          1h
redis-4282552436-neov9       1/1       Running   0          1h
```


# 参考
* [部署分布式kubernetes（v1.1.1）-centos7](http://www.pangxie.space/docker/118)
