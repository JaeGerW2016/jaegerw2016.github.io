---
title:  Kubernetes 节点OutOfDisk 状态Unknown问题排查
date: 2019-08-14 10:53
tag: kubernetes
---



```shell
内核版本: 4.15.0-54-generic 
操作系统镜像:Ubuntu 18.04 LTS 
容器运行时版本: docker://18.9.2 
Kubelet 版本: v1.13.3 
Kube-Proxy 版本: v1.13.3 
操作系统: linux 
架构: amd64 
```

在一次登录Dashboard的查看节点资源使用情况的时候，偶然看到一个Condition的指标OutOfDisk的状态从2019年2月份的时候就开始一直是`Unknown`状态

接下来就是各种查看日志, 查到kubernetes的`issues`  `PR`

https://github.com/kubernetes/kubernetes/pull/70111/files

https://github.com/kubernetes/kubernetes/issues/72485

由于这个Condition的指标是由`Kubelet`的`pkg/kubelet/kubelet_node_status.go`里实现

```go
...
// setNodeStatus fills in the Status fields of the given Node, overwriting
// any fields that are currently set.
// TODO(madhusudancs): Simplify the logic for setting node conditions and
// refactor the node status condition code out to a different file.
func (kl *Kubelet) setNodeStatus(node *v1.Node) {
	for i, f := range kl.setNodeStatusFuncs {
		klog.V(5).Infof("Setting node status at position %v", i)
		if err := f(node); err != nil {
			klog.Warningf("Failed to set some node status fields: %s", err)
		}
	}
}

// updateNodeStatus updates node status to master with retries if there is any
// change or enough time passed from the last sync.
func (kl *Kubelet) updateNodeStatus() error {
	klog.V(5).Infof("Updating node status")
	for i := 0; i < nodeStatusUpdateRetry; i++ {
		if err := kl.tryUpdateNodeStatus(i); err != nil {
			if i > 0 && kl.onRepeatedHeartbeatFailure != nil {
				kl.onRepeatedHeartbeatFailure()
			}
			klog.Errorf("Error updating node status, will retry: %v", err)
		} else {
			return nil
		}
	}
	return fmt.Errorf("update node status exceeds retry count")
}

// defaultNodeStatusFuncs is a factory that generates the default set of
// setNodeStatus funcs
func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
	// if cloud is not nil, we expect the cloud resource sync manager to exist
	var nodeAddressesFunc func() ([]v1.NodeAddress, error)
	if kl.cloud != nil {
		nodeAddressesFunc = kl.cloudResourceSyncManager.NodeAddresses
	}
	var validateHostFunc func() error
	if kl.appArmorValidator != nil {
		validateHostFunc = kl.appArmorValidator.ValidateHost
	}
	var setters []func(n *v1.Node) error
	setters = append(setters,
		nodestatus.NodeAddress(kl.nodeIP, kl.nodeIPValidator, kl.hostname, kl.hostnameOverridden, kl.externalCloudProvider, kl.cloud, nodeAddressesFunc),
		nodestatus.MachineInfo(string(kl.nodeName), kl.maxPods, kl.podsPerCore, kl.GetCachedMachineInfo, kl.containerManager.GetCapacity,
			kl.containerManager.GetDevicePluginResourceCapacity, kl.containerManager.GetNodeAllocatableReservation, kl.recordEvent),
		nodestatus.VersionInfo(kl.cadvisor.VersionInfo, kl.containerRuntime.Type, kl.containerRuntime.Version),
		nodestatus.DaemonEndpoints(kl.daemonEndpoints),
		nodestatus.Images(kl.nodeStatusMaxImages, kl.imageManager.GetImageList),
		nodestatus.GoRuntime(),
	)
	if utilfeature.DefaultFeatureGate.Enabled(features.AttachVolumeLimit) {
		setters = append(setters, nodestatus.VolumeLimits(kl.volumePluginMgr.ListVolumePluginWithLimits))
	}
	setters = append(setters,
		nodestatus.MemoryPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderMemoryPressure, kl.recordNodeStatusEvent),
		nodestatus.DiskPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderDiskPressure, kl.recordNodeStatusEvent),
		nodestatus.PIDPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderPIDPressure, kl.recordNodeStatusEvent),
		nodestatus.ReadyCondition(kl.clock.Now, kl.runtimeState.runtimeErrors, kl.runtimeState.networkErrors, kl.runtimeState.storageErrors, validateHostFunc, kl.containerManager.Status, kl.recordNodeStatusEvent),
		nodestatus.VolumesInUse(kl.volumeManager.ReconcilerStatesHasBeenSynced, kl.volumeManager.GetVolumesInUse),
		// TODO(mtaufen): I decided not to move this setter for now, since all it does is send an event
		// and record state back to the Kubelet runtime object. In the future, I'd like to isolate
		// these side-effects by decoupling the decisions to send events and partial status recording
		// from the Node setters.
		kl.recordNodeSchedulableEvent,
	)
	return setters
}
...
```

看这部分源码的`setNodeStatus`的默认工厂函数`defaultNodeStatusFuncs`中对`Setter`的状态相关condition追加，返回node的包含这些Condition的一个setters的一个集合

`setNodeStatus` 这个函数在遍历这些Condition的过程中，如果碰到具体的error 就输出error日志

`pkg/kubelet/nodestatus/setters.go`

````
...

// Setter modifies the node in-place, and returns an error if the modification failed.
// Setters may partially mutate the node before returning an error.
type Setter func(node *v1.Node) error

...
````

### 结论

由于`OutOfDiskCondition` 由于版本1.12版本迭代到1.13过程中，社区将这个Condition的移除了，由于跟`DiskPressureCondition` 的状态重叠，因此在1.13版本之后可以忽略这个指标



### 

排查过程中，对kubelet在跟apiserver之间的访问 偶然出现Node 状态`NotReady`的问题也进行 优化：

- 磁盘i/o timeout ：调整部分日志级别、减少冗余日志输出
- 大量 ConfigMap/Secret 导致Kubernetes缓慢 :  针对1.13版本的调整apiserver的启动参数 `--http2-max-streams-per-connection` 到一个较大值，如果要彻底修复就得 升级 Go 版本到 1.12 后重新构建 Kubernetes（社区正在进行中）
- 当node 数量到达一定数量，kebulet与apiserver之间访问超时 不可访问：调整apiserver的启动参数`--max-requests-inflight=3000`  `--max-mutating-requests-inflight=1000`



```
 --max-mutating-requests-inflight int                      在给定时间内进行中可变请求的最大数量。当超过该值时，服务将拒绝所有请求。0值表示没有限制。（默认值200）

      --max-requests-inflight int                               在给定时间内进行中不可变请求的最大数量。当超过该值时，服务将拒绝所有请求。0值表示没有限制。（默认值400）
```

### 参考文档

https://feisky.gitbooks.io/kubernetes/troubleshooting/cluster.html

https://github.com/kubernetes/kubernetes/issues/60042

https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/

https://github.com/kubernetes/sig-release/blob/master/releases/release-1.9/scalability_validation_report.md

https://github.com/kubernetes/kubernetes/issues/74302

