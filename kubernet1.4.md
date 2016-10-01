
# kubernetes 1.4 新特性

## 部署一个集群变的及其简单！

kubernetes 1.4 引入了新的kubeadm命令，通过这个命令可以方便快速的创建一个集群。这个命令会帮助我们创建集群所需要的证书、static pods等配置文件。然后通过kubelet以容器的方式启动kube-api-server、kube-controller-manager、kube-proxy、etcd等组件。

但请注意，kubelet启动static pod时会检查主机的hostname `must match the regex [a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)* (e.g. 'example.com')]`。并且官方的指导文档里现在遗漏了一点：没有说明ubuntu下需要先启动kubelet服务:)

## 有状态应用的支持

## 扩展对集群联邦的支持

## 容器安全

## 基础能力的提升


# 参考
* [
Kubernetes 1.4发布：让Kubernetes能在任何地方更简单地运行](http://dockone.io/article/1717)

