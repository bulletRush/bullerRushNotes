# 关注点
* 新的集群部署模型：kubernetes组件及插件pod化；kubeadm工具快速搭建集群
* 支持自动创建云盘：新增StorageClass
* 功能继续向api server收缩：部分object现在由api server设置默认参数，而不是kubelet
* 新增定时任务支持：新增ScheduledJob
* Pod支持sysctl：可以设置白名单
* 调度系统优化：kubelet主动驱逐pod；优先保证基础组件pod运行；并发控制

# 新特性

## ** API Machinery 

### `alpha` AUDIT日志

kube-apiserver添加了AUDIT日志，可以根据audit日志查看用户或者组件什么时间、在那台机器上对集群做了什么操作

### `beta` 支持swagger 2.0

### `beta` [Garbage Collection (Beta)](http://kubernetes.io/docs/user-guide/garbage-collection/)
默认开启server端GC：删除RC或ReplicaSet时自动回收对应的pod。

* **TODO**：需测试如何使用以及对哪些资源生效

## ** Apps

### `alpha` 添加[Scheduled Jobs](http://kubernetes.io/docs/user-guide/scheduled-jobs/)
类似于crontab，定时任务

## ** Auth

### `alpha` 可以限制容器拉取的频率了？ 
Container Image Policy allows an access controller to determine whether a pod may be scheduled based on a policy. [ImagePolicyWebhook](http://kubernetes.io/docs/admin/admission-controllers/#imagepolicywebhook)

* **TODO**: 需测试如何使用

### `alpha` 提供了创建、删除的权限检查接口Checking API Accesshttp://kubernetes.io/docs/admin/authorization/)
 Access Review APIs expose authorization engine to external inquiries for delegation, inspection, and debugging

## ** Cluster Lifecycle

### `alpha` 优先保证集群所必须的基础pod（heapster/dns）可以被创建出来
[“Guaranteed” scheduling of critical add-on pods](http://kubernetes.io/docs/admin/rescheduler/#guaranteed-scheduling-of-critical-add-on-pods)

### `alpha` Simplifies bootstrapping of TLS secured communication between the API server and kubelet. 
[Kubelet TLS Bootstrap](http://kubernetes.io/docs/admin/master-node-communication/#kubelet-tls-bootstrap)

### `alpha` 新增kubeadm工具，快速创建集群

通过这个命令可以方便快速的创建一个集群。这个命令会帮助我们创建集群所需要的证书、static pods等配置文件。然后通过kubelet以容器的方式启动kube-apiserver、kube-controller-manager、kube-proxy、etcd等组件。

但请注意，kubelet启动static pod时会检查主机的hostname `must match the regex [a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)* (e.g. 'example.com')]`。

## ** Federation

略

## ** Network

### `alpha` Service LB支持保留源IP？
Service LB now has alpha support for preserving client source IP. [Creating an External Load Balancer](http://kubernetes.io/docs/user-guide/load-balancer/)

## ** Node

### `alpha` [Node性能测试](http://node-perf-dash.k8s.io/#/builds)
* 看上去像是一个模拟器的东东。。。。展示了pod创建时间、cpu/内存使用率等数据，建议体验一下。

### `alpha` Pod可以执行sysctl修改内核参数了
* [how sysctls are used within a Kubernetes cluster](http://kubernetes.io/docs/admin/sysctls/)
* 对于不安全的sysctl，可以设置白名单。

### `beta`  AppArmor profiles can be specified & applied to pod containers
* **TODO**

### `beta` Cluster policy to control access and defaults of security related features
* **TODO**

### `stable` 当磁盘空间存在压力时，kubelet可以驱逐pod了
* [Configuring Out Of Resource Handling](http://kubernetes.io/docs/admin/out-of-resource/)

### `stable`  Automated docker validation results posted to https://k8s-testgrid.appspot.com/docker

* **TODO**：这是啥？

## ** Scheduling

### `alpha` 允许禁止并发调度pod
* Allows pods to require or prohibit (or prefer or prefer not) co-scheduling on the same node (or zone or other topology domain) as another set of pods.
*  [Assigning Pods to Nodes](http://kubernetes.io/docs/user-guide/node-selection/)
* **TODO**：需要测试体验

## ** Storage
 
### 添加StorageClass
 
见[Persistent Volumes](http://kubernetes.io/docs/user-guide/persistent-volumes/)。这个新的object用于动态的创建PV，简单来说就是k8s现在可以动态创建共享存储了。
 
* **TODO**: PV的guide文档中有讲到k8s会保证用户可以得到想要的PV，但有可能会多创建一些？` but the volume may be in excess of what was requested.`
* **TODO**： Volumes that were dynamically provisioned are always deleted，动态创建的PV在不使用的时候会被销毁？不合理不安全
* **TODO**：云硬盘不支持Retain和Recycle？
* **TODO**: 包年包月方式购买云硬盘怎么续费？

### `stable` 添加了两个新的volume插件（Azure和Quobyte Distributed File System）

## ** UI

###  `stable` Kubernetes Dashboard UI改进

### `stable` kubelet不会设置object的默认值，改为api server设置


# 已知问题

**TODO**

# 不兼容改造

**TODO**

# 升级要求

*TODO*

# 参考
* [Kubernetes 1.4发布：让Kubernetes能在任何地方更简单地运行](http://dockone.io/article/1717)
* [Audit in Kubernetes](http://kubernetes.io/docs/admin/audit/)
* [Persistent Volumes](http://kubernetes.io/docs/user-guide/persistent-volumes/)
