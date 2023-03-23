---
title: 专用的Node自定义准入控制器Dedicate-toleration-webhook
date: 2019-10-18 14:53
tag: kubernetes
---

如果您想要设置一些 node 为专用的，以让特定的用户才能使用，或者具有特定标签的应用使用，这里就需要涉及Taints和Toleration来引到Pod远离pod或者调度具有特定标签的应用pod到Node上



### 一、Node添加Taints

```
kubectl taint nodes nodename dedicated=groupName:NoSchedule
```

### 二、部署Dedicated-toleration-webhook

```shell
git@github.com:JaeGerW2016/dedicated-toleration-webhook.git
cd deployment

# namespace: default service: dedicated-toleration-webhook-service
bash create-signed-cert.sh
bash patch-ca-bundle.sh

cd ../k8s
# Modify env according to your needs in deployment.yaml
#         env:
#            - name: MATCH_LABEL_KEY
#              value: component
#            - name: MATCH_LABEL_VALUE
#              value: nvidia-gpu
#            - name: TOLERATION_KEY
#              value: dedicated
#            - name: TOLERATION_OPERATOR
#              value: Equal
#            - name: TOLERATION_VALUE
#              value: gpu
#            - name: TOLERATION_EFFECT
#              value: NoSchedule
#kustomize build to apply 

kustomize build ../k8s/ | kubectl apply -f -
```

### 三、示例验证

#### pod.yaml

```
apiVersion: v1
kind: Pod
metadata:
 name: gpu-pod
 labels:
   component: nvidia-gpu
spec:
 containers:
   - name: gpu-container
     image: mirrorgcrio/pause:2.0
 nodeSelector:
   component: nvidia-gpu
```

#### deployment.yaml

```
apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: gpu-container-deployment
  labels:
    component: nvidia-gpu
spec:
  replicas: 1
  template:
    metadata:
      labels:
        component: nvidia-gpu
    spec:
      containers:
      - name: gpu-container
        image: mirrorgcrio/pause:2.0
      nodeSelector:
        component: nvidia-gpu
```

#### Taints node 192.168.2.230

```
kubectl taint node 192.168.2.230 dedicated=gpu:NoSchedule
kubectl apply -f pod.yaml 
kubectl apply -f deployment.yaml
```

#### OUTPUT

```
Name:               gpu-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               192.168.2.230/192.168.2.230
Start Time:         Thu, 17 Oct 2019 09:40:05 +0000
Labels:             component=nvidia-gpu
Annotations:        kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"component":"nvidia-gpu"},"name":"gpu-pod","namespace":"default"},"...
Status:             Running
IP:                 172.20.1.7
Containers:
  gpu-container:
    Container ID:   docker://dbf263155bd02a058adc2dcbbdff97a18075075bb9599d06dd4e19340fff8ef8
    Image:          mirrorgcrio/pause:2.0
    Image ID:       docker-pullable://mirrorgcrio/pause@sha256:7b8e6ddd6aa2da43ecad8997e6a984616f4de4727195e48eae2019c1975a78ef
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 17 Oct 2019 09:40:06 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-wnlk9 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-wnlk9:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-wnlk9
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  component=nvidia-gpu
Tolerations:     dedicated=gpu:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>

```

```yaml
Name:               gpu-container-deployment-545b966d9d-x4727
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               192.168.2.230/192.168.2.230
Start Time:         Thu, 17 Oct 2019 09:46:47 +0000
Labels:             component=nvidia-gpu
                    pod-template-hash=545b966d9d
Annotations:        <none>
Status:             Running
IP:                 172.20.1.9
Controlled By:      ReplicaSet/gpu-container-deployment-545b966d9d
Containers:
  gpu-container:
    Container ID:   docker://107df845330447c8c5438e65628dcf79b571a5a1477d18084fef3f7c4398a533
    Image:          mirrorgcrio/pause:2.0
    Image ID:       docker-pullable://mirrorgcrio/pause@sha256:7b8e6ddd6aa2da43ecad8997e6a984616f4de4727195e48eae2019c1975a78ef
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 17 Oct 2019 09:46:48 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-wnlk9 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-wnlk9:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-wnlk9
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  component=nvidia-gpu
Tolerations:     dedicated=gpu:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```





### 参考文档：

https://k8smeetup.github.io/docs/concepts/configuration/assign-pod-node/

https://github.com/aflc/extended-resource-toleration-webhook

https://github.com/JaeGerW2016/dedicated-toleration-webhook