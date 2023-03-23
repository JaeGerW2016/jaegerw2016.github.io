---
title:  KubernetesDeployment滚动更新机制
date: 2019-09-27 14:53
tag: kubernetes
---



## 摘要

Kubernetes Deployment滚动更新机制，提供了滚动进度查询，滚动历史记录，回滚等能力，无疑是使用Kubernetes进行应用滚动发布的首选



## rolling update的相关项

以下面的frontend Deployment为例

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: frontend
spec:
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 2
  replicas: 6
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80
```

针对这个yml文件中滚到更新的部分

```
...
spec:
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 2
...
```

- `.spec.minReadySeconds`: is an optional field that specifies the minimum number of seconds for which a newly created Pod should be ready without any of its containers crashing, for it to be considered available. This defaults to 0 (the Pod will be considered available as soon as it is ready). To learn more about when a Pod is considered ready  新生成的Pod状态为Ready持续的时间至少为`.spec.minReadySeconds`才认为Pod Available(Ready)，默认秒数为0。
- `.spec.strategy.rollingUpdate.maxSurge` is an optional field that specifies the maximum number of Pods that can be created over the desired number of Pods. The value can be an absolute number (for example, 5) or a percentage of desired Pods (for example, 10%). The value cannot be 0 if `MaxUnavailable` is 0. The absolute number is calculated from the percentage by rounding up. The default value is 25%. 最大允许超出`.spec.replicas`的pod数量，这个可以是整数（例如：5）或者百分比（例如：10%），这个 参数的值不可为0，如果`MaxUnavailable`为0，按照百分比计算的得到的绝对值，进行向上取整，默认值为25%
- `.spec.strategy.rollingUpdate.maxUnavailable` is an optional field that specifies the maximum number of Pods that can be unavailable during the update process. The value can be an absolute number (for example, 5) or a percentage of desired Pods (for example, 10%). The absolute number is calculated from percentage by rounding down. The value cannot be 0 if `.spec.strategy.rollingUpdate.maxSurge` is 0. The default value is 25%. 最大允许Pod不可用的数量，这个值可以指定为整数或者百分比，按照百分比计算得到的绝对值进行向下取整，这个值不可为0，当`.spec.strategy.rollingUpdate.maxSurge` 的值为0时，默认是25%，



因此可以通过这些指标计算得到：

**在Deployment rollout时，需要保证Available(Ready) Pods数不低于 `desired pods replicas - maxUnavailable`; 保证所有的Pods数不多于 `desired pods replicas + maxSurge`。**

## 滚动更新过程

> **Note:** A Deployment’s rollout is triggered if and only if the Deployment’s pod template (that is, `.spec.template`) is changed, for example if the labels or container images of the template are updated. Other updates, such as scaling the Deployment, do not trigger a rollout.

继续以上面的Deployment为例子，并考虑最常用的情况:

`.spec.template.spec.containers[0].image` from `gcr.io/google-samples/gb-frontend:v4` to `gcr.io/google-samples/gb-frontend:v5`

```shell
kubectl set image deploy frontend php-redis=gcr.io/google-samples/gb-frontend:v5 --record
```

set image之后，导致Deployment’s Pod Template hash发生变化，就会触发rollout

过程：

- NewRS创建maxSurge(3)个Pods，这时达到pods数的上限值`desired replicas + maxSurge` (9个）
- 不会等NewRS创建的Pods Ready，而是马上delete OldRS maxUnavailable(2)个Pods，这时Ready的Pods number最差也能保证`desired replicas - maxUnavailable`(4个)
- 接下来的流程是不固定，只要新建的Pods有几个返回Ready，则意味着可以接着删除几个旧的Pods了。只要有几个删除成功的Pods返回，就会创建一定数量的Pods，只要All pods数量与上限值desired replicas + maxSurge有差值空间，就会接着创建新的Pods。
- 如此进行滚动更新， 直到创建的新Pods个数达到`desired replicas`，并等待它们都Ready，然后再删除所有剩余的旧的Pods。至此，滚动流程结束。

## 理解rollout pause和resume

> You can pause a Deployment before triggering one or more updates and then resume it. This will allow you to apply multiple fixes in between pausing and resuming without triggering unnecessary rollouts.

`kubectl rollout pause`只会用来停止触发下一次rollout，正在执行的滚动历程是不会停下来的，而是会继续正常的进行滚动，直到完成。等下一次，用户再次触发rollout时，Deployment就不会真的去启动执行滚动更新了，而是等待用户执行了`kubectl rollout resume`，流程才会真正启动执行。

For example, with a Deployment that was just created:

```shell
$ kubectl get deploy
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     3         3         3            3           1m
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         1m
```

Pause by running the following command:

```shell
$ kubectl rollout pause deployment/nginx-deployment
deployment "nginx-deployment" paused
```

Then update the image of the Deployment:

```shell
$ kubectl set image deploy/nginx-deployment nginx=nginx:1.9.1
deployment "nginx-deployment" image updated
```

Notice that no new rollout started:

```shell
$ kubectl rollout history deploy/nginx-deployment
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>

$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   3         3         3         2m
```

You can make as many updates as you wish, for example, update the resources that will be used:

```shell
$ kubectl set resources deployment nginx -c=nginx --limits=cpu=200m,memory=512Mi
deployment "nginx" resource requirements updated
```

The initial state of the Deployment prior to pausing it will continue its function, but new updates to the Deployment will not have any effect as long as the Deployment is paused.

Eventually, resume the Deployment and observe a new ReplicaSet coming up with all the new updates:

```shell
$ kubectl rollout resume deploy/nginx-deployment
deployment "nginx" resumed
$ kubectl get rs -w
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   2         2         2         2m
nginx-3926361531   2         2         0         6s
nginx-3926361531   2         2         1         18s
nginx-2142116321   1         2         2         2m
nginx-2142116321   1         2         2         2m
nginx-3926361531   3         2         1         18s
nginx-3926361531   3         2         1         18s
nginx-2142116321   1         1         1         2m
nginx-3926361531   3         3         1         18s
nginx-3926361531   3         3         2         19s
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         1         1         2m
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         20s
^C
$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

> **Note:** You cannot rollback a paused Deployment until you resume it

## ReplicaSet和rollout history的关系

> You can set `.spec.revisionHistoryLimit` field in a Deployment to specify how many old ReplicaSets for this Deployment you want to retain. 
>
> Setting the kubectl flag –record to true allows you to record current command in the annotations of the resources being created or updated.

> By default, all revision history will be kept. In a future version, it will default to switch to 2.

默认情况下，所有通过kubectl xxxx –record都会被kubernetes记录到etcd进行持久化，这无疑会占用资源，最重要的是，时间久了，当你kubectl get rs时，会有成百上千的垃圾RS返回给你，那时你可能就眼花缭乱了。

最好通过设置Deployment的`.spec.revisionHistoryLimit`来限制最大保留的revision number，比如15个版本，回滚的时候一般只会回滚到最近的几个版本就足够了。

>**Note:** Explicitly setting this field to 0, will result in cleaning up all the history of your Deployment thus that Deployment will not be able to roll back

## 回滚是如何进行的

用户通过执行rollout undo并指定`--to-revison`，可以将Deployment回滚到指定的revision。

```shell
kubectl rollout undo deploy frontend --to-revision=7
```

回滚的时候也是按照滚动的机制进行的，同样要遵守maxSurge和maxUnavailable的约束。并不是一次性将所有的Pods删除，然后再一次性创建新的Pods。
