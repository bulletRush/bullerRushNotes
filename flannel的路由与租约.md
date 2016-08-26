
flannel的路由注册是在`LocalManager`中的`tryAcquireLease`中完成的。 在这里，flannel会先向etcd查询当前节点（子机）的路由是否已经注册过；在没有注册过的情况下，flannel会在etcd设置的CIDR中随机分配一个子网（如果分配失败，则会自动进行重试）。 在写入etcd时，flannle会设置当前节点的ttl为24小时。 flannel内部将注册的子网称为一个`lease`（租约）。 在租约到期的前一个小时，flannel会尝试向etcd续约。

但是需要注意的是，如果flannel节点与etcd server的时间相差较大，则有可能导致flannel在没有发起续约时，etcd就清除了对应的‘租约’。因为flannel的续约时间的计算方式是这样的：

```go
// network.go:Network.runOnce
	// 这里的Expiration是etcd server的过期时间，renewMargin是一个小时
	// 如果flannel所在节点时间比etcd server慢一个小时，则有可能导致续约前etcd就清除了对应的key信息
	dur := n.bn.Lease().Expiration.Sub(time.Now()) - renewMargin
	for {
		select {
		case <-time.After(dur):
			err := n.sm.RenewLease(n.ctx, n.Name, n.bn.Lease())
			if err != nil {
				log.Error("Error renewing lease (trying again in 1 min): ", err)
				dur = time.Minute
				continue
			}

			log.Info("Lease renewed, new expiration: ", n.bn.Lease().Expiration)
			dur = n.bn.Lease().Expiration.Sub(time.Now()) - renewMargin

```

这是在帮一个群友查看某台机器flannel路由总是莫名失效时发现的一个问题，他的实际解决方法是同步了一下服务器的时间，但是由于他并没有注意同步前服务器时间相差多少，同时，我也没有拿到有用的日志信息，因此并不确定问题是否与这里的逻辑有关。
