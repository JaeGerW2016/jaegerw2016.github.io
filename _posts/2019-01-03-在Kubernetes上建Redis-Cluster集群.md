---
title: 在Kubernetes上建Redis Cluster集群
date: 2019-01-03 02:37:45
tags:
---
###文档说明
实验环境：kubernetes Version v1.9.6
网络CNI：fannel
存储CSI: NFS Dynamic Class

####前期准备
利用NFS动态提供Kubernetes后端存储卷[https://jimmysong.io/kubernetes-handbook/practice/using-nfs-for-persistent-storage.html]
构建含有peer-finder的redis:4.0.11镜像（googleapi.com国内无法访问） https://hub.docker.com/r/314315960/redis-peer-finder/
`docker pull 314315960/redis-peer-finder:4.0.11`
构建redis集群管理工具redis-trib 镜像 https://hub.docker.com/r/314315960/redis-trib/
`docker pull 314315960/redis-trib`



一、创建pvc
`redis-pvc.yaml`
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "default"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

二、创建configmap
`redis-cm.yaml`
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-conf
data:
  redis.conf: |
          appendonly yes
          cluster-enabled yes
          cluster-require-full-coverage no
          cluster-config-file nodes.conf
          cluster-migration-barrier 1
          cluster-node-timeout 1000
          protected-mode no

  bootstrap-pod.sh: |
    #!/bin/sh
    set -e
    # Find which member of the Stateful Set this pod is running
    # e.g. "redis-cluster-0" -> "0"
    PET_ORDINAL=$(cat /etc/podinfo/pod_name | rev | cut -d- -f1)
    redis-server /etc/redis/redis.conf &
    # Discover peers (peer-finder构建在镜像里)
    #wget https://storage.googleapis.com/kubernetes-release/pets/peer-finder -O /bin/peer-finder
    chmod u+x /bin/peer-finder
    peer-finder -on-change 'tee > /tmp/initial_peers' -on-start 'tee > /tmp/initial_peers' -service redis-cluster -ns $POD_NAMESPACE
    # TODO: Wait until redis-server process is ready
    sleep 1
    if [ $PET_ORDINAL = "0" ]; then
      # The first member of the cluster should control all slots initially
      redis-cli cluster addslots $(seq 0 16383)
    else
      # Other members of the cluster join as slaves
      # TODO: Get list of peers using the peer finder using an init container
      PEER_IP=$(perl -MSocket -e 'print inet_ntoa(scalar(gethostbyname("redis-cluster-0.redis-cluster.default.svc.cluster.local")))')
      redis-cli cluster meet $PEER_IP 6379
      sleep 1
      #echo redis-cli --csv cluster slots
      #redis-cli --csv cluster slots
      # Become the slave of a random master node
      MASTER_ID=$(redis-cli --csv cluster slots | cut -d, -f 5 | sed -e 's/^"//'  -e 's/"$//')
      redis-cli cluster replicate $MASTER_ID
    fi
    wait

```

三、StatefulSet
`redis-statefulset.yaml`
```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: redis-cluster
  replicas: 6
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis-cluster
        image: 314315960/redis-peer-finder:4.0.11
        ports:
        - containerPort: 6379
          name: redis
        - containerPort: 16379
          name: gossip
        command:
        - "/bin/sh"
        args:
        - "/etc/redis/bootstrap-pod.sh"
        readinessProbe:
          exec:
            command:
            - "/bin/sh"
            - "-c"
            - "redis-cli -h $(hostname) ping"
          initialDelaySeconds: 15
          timeoutSeconds: 15
        livenessProbe:
          exec:
            command:
            - "/bin/sh"
            - "-c"
            - "redis-cli -h $(hostname) ping"
          initialDelaySeconds: 20
          periodSeconds: 3
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: redis-conf
          mountPath: /etc/redis
          readOnly: false
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
        - name: redis-data
          mountPath: /var/lib/redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: redis-conf
        configMap:
          name: redis-conf
          items:
            - key: redis.conf
              path: redis.conf
            - key: bootstrap-pod.sh
              path: bootstrap-pod.sh
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-claim
      - name: podinfo
        downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "annotations"
              fieldRef:
                fieldPath: metadata.annotations
            - path: "pod_name"
              fieldRef:
                fieldPath: metadata.name
            - path: "pod_namespace"
              fieldRef:
                fieldPath: metadata.namespace

```
四、Headless Service
`redis-service.yaml`
```
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  labels:
    app: redis
    appCluster: redis-cluster
spec:
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: gossip
    port: 16379
    targetPort: 16379
  clusterIP: None
  selector:
    appCluster: redis-cluster

```
五、运行redis-trib 创建集群
> create 前面的 `--` 非常关键，如果不加的话，`--replicas 1`会被kubectl误认为是kubectl的参数 `--replicas 1`,从而导致redis-trib建的集群为6个master的集群，而不是3master 3slave的集群（这个问题耗费了一个下午排查）
```
kubectl run redis-trib --image=314315960/redis-trib -- create --replicas 1  \
$(kubectl get pod redis-app-0 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-1 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-2 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-3 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-4 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-5 -o jsonpath='{.status.podIP}'):6379
```

验证:
![image.png](https://upload-images.jianshu.io/upload_images/3481257-ca3e4edddf3b9a81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


参考文档：
https://www.jianshu.com/p/65c4baadf5d9
https://github.com/sanderploegsma/redis-cluster
http://weizijun.cn/2016/01/08/redis%20cluster%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7redis-trib-rb%E8%AF%A6%E8%A7%A3/
https://redis.io/topics/cluster-tutorial
https://ask.csdn.net/questions/347220
https://storage.googleapis.com/kubernetes-release/pets/peer-finder
https://github.com/kubernetes/contrib/tree/master/peer-finder
