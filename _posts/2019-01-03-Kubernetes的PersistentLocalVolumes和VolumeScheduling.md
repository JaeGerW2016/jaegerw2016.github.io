---
title: Kubernetes的PersistentLocalVolumes和VolumeScheduling
date: 2019-01-03 02:48:05
tags:
---

### 文档说明
实验环境：kubernetes Version v1.9.6
网络CNI：fannel


#### 本地卷：
更好的利用本地高性能介质（SSD，Flash）提升数据库服务能力 QPS/TPS
更闭环的运维成本，现在越来越多的数据库支持基于Replicated的技术实现数据多副本和数据一致性（比如MySQL Group Replication / MariaDB Galera Cluster / Percona XtraDB Cluster的），DBA可以处理所有问题，而不在依赖存储工程师或者SA的支持。


>为了使用本地存储需要启动FeatureGate:PersistentLocalVolumes支持本地存储，1.9是alpha版本，1.10是beta版，默认开启, v1.9版本需要api-server, controller-manager, scheduler, and all kubelets 开启 feature-gates的功能
```
--feature-gates=PersistentLocalVolumes=true
--VolumeScheduling=true
--MountPropagation=true
```

#### 实战示例：
一、创建`PersistentVolume`
```
apiVersion: v1
kind: PersistentVolume
metadata:
name: local-pv
annotations:
    "volume.alpha.kubernetes.io/node-affinity": '{
        "requiredDuringSchedulingIgnoredDuringExecution": {
            "nodeSelectorTerms": [
                { "matchExpressions": [
                    { "key": "kubernetes.io/hostname",
                      "operator": "In",
                      "values": ["k8s-node1-product"]
                    }
                ]}
             ]}
          }'
spec:
capacity:
  storage: 10Gi
accessModes:
- ReadWriteOnce
persistentVolumeReclaimPolicy: Delete
storageClassName: local-storage
local:
  path: /mnt/disks/ssd1
```
二、创建`Storage Class`
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
三、创建`Statefulset`
```
apiVersion: apps/v1beta1 
kind: StatefulSet 
metadata: 
name: local-test 
spec: 
  serviceName: "local-service" 
  replicas: 1 
  template: 
    metadata: 
      labels: 
        app: local-test 
    spec: 
      containers: 
      - name: test-container 
        image: [gcr.io/google_containers/busybox:1.24]
        command: 
        - "/bin/sh" 
        args: 
        - "-c" 
        - "count=0; count_file=\"/usr/test-pod/count\"; test_file=\"/usr/test-pod/test_file\"; if [ -e $count_file ]; then count=$(cat $count_file); fi; echo $((count+1)) > $count_file; while [ 1 ]; do date >> $test_file; echo \"This is $MY_POD_NAME, count=$(cat $count_file)\" >> $test_file; sleep 10; done" 
        volumeMounts: 
        - name: local-vol 
          mountPath: /usr/test-pod 
        env: 
        - name: MY_POD_NAME 
         valueFrom: 
           fieldRef: 
             fieldPath: metadata.name 
     securityContext: 
       fsGroup: 1234 
 volumeClaimTemplates: 
 - metadata:
   annotations:
     volume.alpha.kubernetes.io/storage-class: local-storage
   name: local-vol 
   spec: 
     accessModes: [ "ReadWriteOnce" ] 
     storageClassName: "local-storage" 
     resources: 
       requests: 
         storage: 10Gi
```
>该Statefulset的Pod将会调度到k8s-node1-product，并使用本地存储“local-pv”


#### “PersistentLocalVolumes”和“VolumeScheduling”的局限


##### 使用局限需要考虑：
###### 具体部署时,针对PersistentLocalVolumes 只能应用在特定的有状态服务的场景下
- 资源利用率降低。一旦本地存储使用完，即使CPU、Memory剩余再多，该节点也无法提供服务；
- 需要做好本地存储规划，譬如每个节点Volume的数量、容量等，就像原来使用存储时需要把LUN规划好一样，在一个大规模运行的环境，存在落地难度。

##### 高可用风险需要考虑：

当Pod调度到某个节点后，将会跟该节点产生亲和，一旦Node发生故障，Pod不能调度到其他节点，只能等待该节点恢复，你能做的就是等待“Node恢复”，如果部署3节点MySQL集群，再挂一个Node，集群将无法提供服务，你能做的还是“等待Node恢复”。这么设计也是合理的，社区认为该Node为Stateful节点，Pod被调度到其他可用Node会导致数据丢失

>最后，借用Google工程师Kelsey Hightower的一句话:
“We very receptive this Kubernetes can’t be everything to everyone.”


参考文档：
http://dockone.io/article/5260
https://blog.zhoulouzi.com/2018/03/kubernetes-local/
https://www.kubernetes.org.cn/2280.html?tdsourcetag=s_pctim_aiomsg
https://kubernetes.io/docs/concepts/storage/volumes/#local

