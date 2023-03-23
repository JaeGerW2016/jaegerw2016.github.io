---
title: Prometheus Operator 学习笔记（二）Service Endpoints 重置问题
date: 2019-01-07 02:24:10
tags:
---
### 文档说明
实验环境：kubernetes Version v1.10.9
网络CNI：fannel
存储CSI: NFS Dynamic Class
DNS: CoreDNS

### 背景 
 部署完Prometheus Operator之后 在Prometheus的 Alert监控事项中会收到 Kube-scheduler和kube-controller-manager的告警信息
`KubeSchedulerDown`
```
alert: KubeSchedulerDown
expr: absent(up{job="kube-scheduler"}
  == 1)
for: 15m
labels:
  severity: critical
annotations:
  message: KubeScheduler has disappeared from Prometheus target discovery.
  runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubeschedulerdown
```
`KubeControllerManagerDown`
```
alert: KubeControllerManagerDown
expr: absent(up{job="kube-controller-manager"}
  == 1)
for: 15m
labels:
  severity: critical
annotations:
  message: KubeControllerManager has disappeared from Prometheus target discovery.
  runbook_url: https://github.com/kubernetes-monitoring/kubernetes-mixin/tree/master/runbook.md#alert-name-kubecontrollermanagerdown
```
#### 查到问题是由于`kube-scheduler`和`kube-controller-manager`的Endpoints地址被重置成`none`导致的

接下来就开启了查原因的漫漫之路，困扰了一个星期

- kube-scheduler和kube-controller-manager 的启动参数
- Kubernetes Endpoints Controller源码分析
- service 和 endpoints 官方文档定义 

### [kube-controller-manager 的启动参数](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

参考了[XuXinkun Blog](https://xuxinkun.github.io/2017/09/18/kube-endpoint-disappear/) 的这篇博客的问题分析定位，我查看了集群位于`/var/log/syslog`下的kubernetes的日志，发现了同样的NoteReady日志输出，认为*kube-controller-manager判断node上报心跳超时的时间默认为40秒*，存在一定几率的超时导致，所以一开始以为找到了问题的原因，立马参照他的解决方法调整`kube-scheduler.service`的启动参数`--node-monitor-grace-period duration=60s`
```
--node-monitor-grace-period duration     Default: 40s
Amount of time which we allow running Node to be unresponsive before marking it unhealthy. Must be N times more than kubelet's nodeStatusUpdateFrequency, where N means number of retries allowed for kubelet to post node status.
```
> 观察重新apply -f endpoints文件之后，还是存在endpoints变成none，问题还是存在！！！

起码知道了可能导致这个问题的原因，就继续查问题，还是通过查看了集群位于`/var/log/syslog`下的kubernetes的日志，过滤每一条有价值的日志，发现日志的信息中Timeout的时长常常维持在7-9分钟这样一个期间，所以我索性改`--node-monitor-grace-period duration=600s` 这样就不存在`node上报心跳超时`问题

> 观察重新apply -f endpoints文件之后，还是存在endpoints变成none，问题还是存在！！！ 感觉整个人都怀疑人生了

期间不停的google相关的资料，看到了[修复 Service Endpoint 更新的延迟](https://www.yangcs.net/posts/kubernetes-fixing-delayed-service-endpoint-updates/) 这篇博客，又有新的线索，这篇博客中，貌似是更新延迟的问题，顺便也对这个Endpoints更新的机制做了了解，还把集群
kube-controller-manager 的启动参数`--kube-api-qps 和 --kube-api-burst` 改大`--kube-api-qps=300`和`--kube-api-burst=325`和`--concurrent-endpoints-syncs=30`
```
--concurrent-endpoint-syncs int32     Default: 5
The number of endpoint syncing operations that will be done concurrently. Larger number = faster endpoint updating, but more CPU (and network) load
--kube-api-qps float32     Default: 20
QPS to use while talking with kubernetes apiserver.
--kube-api-burst int32     Default: 30
Burst to use while talking with kubernetes apiserver.
```
> 观察重新apply -f endpoints文件之后，还是存在endpoints变成none，问题还是存在！！！ 又失去了问题的线索

###  [Kubernetes Endpoints Controller源码分析](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/endpoint/endpoints_controller.go)

`endpoints_controller.go` 的核心逻辑syncService
```
func (e *EndpointController) syncService(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing service %q endpoints. (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	service, err := e.serviceLister.Services(namespace).Get(name)
	if err != nil {
		// Delete the corresponding endpoint, as the service has been deleted.
		// TODO: Please note that this will delete an endpoint when a
		// service is deleted. However, if we're down at the time when
		// the service is deleted, we will miss that deletion, so this
		// doesn't completely solve the problem. See #6877.
		err = e.client.CoreV1().Endpoints(namespace).Delete(name, nil)
		if err != nil && !errors.IsNotFound(err) {
			return err
		}
		return nil
	}

	if service.Spec.Selector == nil {
		// services without a selector receive no endpoints from this controller;
		// these services will receive the endpoints that are created out-of-band via the REST API.
		return nil
	}

	klog.V(5).Infof("About to update endpoints for service %q", key)
	pods, err := e.podLister.Pods(service.Namespace).List(labels.Set(service.Spec.Selector).AsSelectorPreValidated())
	if err != nil {
		// Since we're getting stuff from a local cache, it is
		// basically impossible to get this error.
		return err
	}

	// If the user specified the older (deprecated) annotation, we have to respect it.
	tolerateUnreadyEndpoints := service.Spec.PublishNotReadyAddresses
	if v, ok := service.Annotations[TolerateUnreadyEndpointsAnnotation]; ok {
		b, err := strconv.ParseBool(v)
		if err == nil {
			tolerateUnreadyEndpoints = b
		} else {
			utilruntime.HandleError(fmt.Errorf("Failed to parse annotation %v: %v", TolerateUnreadyEndpointsAnnotation, err))
		}
	}

	subsets := []v1.EndpointSubset{}
	var totalReadyEps int = 0
	var totalNotReadyEps int = 0

	for _, pod := range pods {
		if len(pod.Status.PodIP) == 0 {
			klog.V(5).Infof("Failed to find an IP for pod %s/%s", pod.Namespace, pod.Name)
			continue
		}
		if !tolerateUnreadyEndpoints && pod.DeletionTimestamp != nil {
			klog.V(5).Infof("Pod is being deleted %s/%s", pod.Namespace, pod.Name)
			continue
		}

		epa := *podToEndpointAddress(pod)

		hostname := pod.Spec.Hostname
		if len(hostname) > 0 && pod.Spec.Subdomain == service.Name && service.Namespace == pod.Namespace {
			epa.Hostname = hostname
		}

		// Allow headless service not to have ports.
		if len(service.Spec.Ports) == 0 {
			if service.Spec.ClusterIP == api.ClusterIPNone {
				subsets, totalReadyEps, totalNotReadyEps = addEndpointSubset(subsets, pod, epa, nil, tolerateUnreadyEndpoints)
				// No need to repack subsets for headless service without ports.
			}
		} else {
			for i := range service.Spec.Ports {
				servicePort := &service.Spec.Ports[i]

				portName := servicePort.Name
				portProto := servicePort.Protocol
				portNum, err := podutil.FindPort(pod, servicePort)
				if err != nil {
					klog.V(4).Infof("Failed to find port for service %s/%s: %v", service.Namespace, service.Name, err)
					continue
				}

				var readyEps, notReadyEps int
				epp := &v1.EndpointPort{Name: portName, Port: int32(portNum), Protocol: portProto}
				subsets, readyEps, notReadyEps = addEndpointSubset(subsets, pod, epa, epp, tolerateUnreadyEndpoints)
				totalReadyEps = totalReadyEps + readyEps
				totalNotReadyEps = totalNotReadyEps + notReadyEps
			}
		}
	}
	subsets = endpoints.RepackSubsets(subsets)

	// See if there's actually an update here.
	currentEndpoints, err := e.endpointsLister.Endpoints(service.Namespace).Get(service.Name)
	if err != nil {
		if errors.IsNotFound(err) {
			currentEndpoints = &v1.Endpoints{
				ObjectMeta: metav1.ObjectMeta{
					Name:   service.Name,
					Labels: service.Labels,
				},
			}
		} else {
			return err
		}
	}

	createEndpoints := len(currentEndpoints.ResourceVersion) == 0

	if !createEndpoints &&
		apiequality.Semantic.DeepEqual(currentEndpoints.Subsets, subsets) &&
		apiequality.Semantic.DeepEqual(currentEndpoints.Labels, service.Labels) {
		klog.V(5).Infof("endpoints are equal for %s/%s, skipping update", service.Namespace, service.Name)
		return nil
	}
	newEndpoints := currentEndpoints.DeepCopy()
	newEndpoints.Subsets = subsets
	newEndpoints.Labels = service.Labels
	if newEndpoints.Annotations == nil {
		newEndpoints.Annotations = make(map[string]string)
	}

	klog.V(4).Infof("Update endpoints for %v/%v, ready: %d not ready: %d", service.Namespace, service.Name, totalReadyEps, totalNotReadyEps)
	if createEndpoints {
		// No previous endpoints, create them
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Create(newEndpoints)
	} else {
		// Pre-existing
		_, err = e.client.CoreV1().Endpoints(service.Namespace).Update(newEndpoints)
	}
	if err != nil {
		if createEndpoints && errors.IsForbidden(err) {
			// A request is forbidden primarily for two reasons:
			// 1. namespace is terminating, endpoint creation is not allowed by default.
			// 2. policy is misconfigured, in which case no service would function anywhere.
			// Given the frequency of 1, we log at a lower level.
			klog.V(5).Infof("Forbidden from creating endpoints: %v", err)
		}
		return err
	}
	return nil
}
```
 >Service的Add/Update/Delete Event Handler都是将Service Key加入到Queue中，等待worker进行syncService处理,syncService方法的逻辑都是建立在通过LabelSelector进行Pod匹配，将匹配的Pods构建对应的Endpoints Subsets加入到Endpoints中，因此这里会先过滤掉那些没有LabelSelector的Services，而上一篇完Prometheus Operator 之后 在监控二进制组件Kube-scheduler和kube-controller-manager以及后续的etcd集群的时候，由于部署方式采用的是非Pod形式在集群内运行

```
	if service.Spec.Selector == nil {
		// services without a selector receive no endpoints from this controller;
		// these services will receive the endpoints that are created out-of-band via the REST API.
		return nil
	}
```
>注释突然提醒了我，立马查看了我的service的yaml文件，感觉找到了问题的根源：
> 由于非Pod形式在集群内运行，所以sevice的yaml文件就不需要定义selector 去过滤pod的标签

特意去查看了[service的官方文档](https://kubernetes.io/zh/docs/concepts/services-networking/service/)
![](https://upload-images.jianshu.io/upload_images/3481257-cd1af3cc7ee4a5e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以根据ServiceMonitor—> Service—>endpoints(pod) 服务发现机制`labelselector`标签来做关系绑定 就需要做调整，统一把非pod形式的service的selector字段去掉。


> 观察重新apply -f endpoints文件之后，问题解决！！！
