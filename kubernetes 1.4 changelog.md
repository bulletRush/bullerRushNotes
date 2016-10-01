
# 禁止转载 by：w-gh

# kubernetes 1.4 新特性

## 优化用户体验

### 部署一个集群变的及其简单！

kubernetes 1.4 引入了新的kubeadm命令，通过这个命令可以方便快速的创建一个集群。这个命令会帮助我们创建集群所需要的证书、static pods等配置文件。然后通过kubelet以容器的方式启动kube-apiserver、kube-controller-manager、kube-proxy、etcd等组件。

但请注意，kubelet启动static pod时会检查主机的hostname `must match the regex [a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)* (e.g. 'example.com')]`。

### AUDIT日志

kube-apiserver添加了AUDIT日志，可以根据audit日志查看用户或者组件什么时间、在那台机器上对集群做了什么操作

### server-based API defaults

what's this?

## 有状态应用支持

### 添加StorageClass

见[Persistent Volumes](http://kubernetes.io/docs/user-guide/persistent-volumes/)。这个新的object用于动态的创建PV，简单来说就是k8s现在可以动态创建共享存储了。

* **TODO**: PV的guide文档中有讲到k8s会保证用户可以得到想要的PV，但有可能会多创建一些？
* **TODO**： Volumes that were dynamically provisioned are always deleted，动态创建的PV在不使用的时候会被销毁？不合理不安全
* **TODO**：云硬盘不支持Retain和Recycle？
* **TODO**: 包年包月方式购买云硬盘怎么续费？

>  but the volume may be in excess of what was requested.

### 新的volume插件

### 添加[Scheduled Jobs](http://kubernetes.io/docs/user-guide/scheduled-jobs/)
类似于crontab，定时任务

### pod/node亲和性

## 扩展对集群联邦的支持

## 安全性

* 增加pod级别的安全保证
* 增加集群级别的安全性保证

# 参考
* [Kubernetes 1.4发布：让Kubernetes能在任何地方更简单地运行](http://dockone.io/article/1717)
* [Audit in Kubernetes](http://kubernetes.io/docs/admin/audit/)
* [Persistent Volumes](http://kubernetes.io/docs/user-guide/persistent-volumes/)
