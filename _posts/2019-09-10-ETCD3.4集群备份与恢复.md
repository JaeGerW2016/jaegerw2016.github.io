---
title: ETCD3.4集群备份与恢复
date: 2019-09-10 12:35
tag: kubernetes
---

## 概述：

etcd被设计为一个提供高可用、强一致的小型KeyValue数据存储服务，在Kubernetes集群中是一个非常重要的组件，用于保存Kubernetes的网络配置信息和Kubernetes对象的元数据信息，一旦etcd挂了，那么也就意味着整个集群就不可用了，所以在实际情况生产环境中我们会部署多个etcd节点，组成etcd高可用集群，在etcd高可用集群中可挂节点数为(n-1)/2个，比如你有3个节点做etcd-ha那么最多可以挂掉(3-1)/2个节点，虽然有HA集群可以保证一定的高可用，但在实际生产环境中还是建议将etcd数据进行备份，方便在出现问题时进行还原。

```
Enviroment Versiom：
OS：debian 9
docker：18.09.6
Kubernetes：v1.14.6
etcd：v3.4

192.168.2.14 	etcd-01
192.168.2.15 	etcd-02
192.168.2.16 	etcd-03
```

## 备份 etcd 集群

备份 etcd 集群可以通过两种方式完成：etcd 内置快照和卷快照

**etcd 支持内置快照**，因此备份 etcd 集群很容易。快照可以从使用 `etcdctl snapshot save` 命令的活动成员中获取，也可以通过从 etcd [数据目录](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/configuration.md#--data-dir) 复制 `member/snap/db` 文件，该 etcd 数据目录目前没有被 etcd 进程使用。`datadir` 位于 `$DATA_DIR/member/snap/db`。获取快照通常不会影响成员的性能。

```shell
root@debian:~# mkdir -p /var/lib/etcd_backup/
root@debian:~# 
root@debian:/var/lib# ETCDCTL_API=3 etcdctl snapshot  save /var/lib/etcd_backup/etcd_$(date "+%Y%m%d%H%M%S").db
{"level":"info","ts":1568093500.2938907,"caller":"snapshot/v3_snapshot.go:109","msg":"created temporary db file","path":"/var/lib/etcd_backup/etcd_20190910133139.db.part"}
{"level":"warn","ts":"2019-09-10T13:31:40.448+0800","caller":"clientv3/retry_interceptor.go:116","msg":"retry stream intercept"}
{"level":"info","ts":1568093500.4698584,"caller":"snapshot/v3_snapshot.go:120","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
{"level":"info","ts":1568093501.8322825,"caller":"snapshot/v3_snapshot.go:133","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","took":1.535686665}
{"level":"info","ts":1568093501.833143,"caller":"snapshot/v3_snapshot.go:142","msg":"saved","path":"/var/lib/etcd_backup/etcd_20190910133139.db"}
Snapshot saved at /var/lib/etcd_backup/etcd_20190910133139.db
root@debian:/var/lib# cd etcd_backup/
root@debian:/var/lib/etcd_backup# ETCDCTL_API=3 etcdctl --write-out=table snapshot status etcd_20190910133139.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| dd8d9bc7 |  1325029 |       1101 |     7.3 MB |
+----------+----------+------------+------------+

```

`etcd_backup.sh`

```
#!/bin/bash
# etcd v3.4 backup script
# create: 2019/09/10 11:00

BACKUP_DIR="/var/lib/etcd_backup"
ENDPOINTS='https://127.0.0.1:2379'
DATA=`date +%Y%m%d%H%M`
export ETCDCTL_API=3

etcdctl --endpoints=$ENDPOINTS \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem \
--cacert=/etc/etcd/ssl/etcd-ca.pem   snapshot save  $BACKUP_DIR/etcd_snapshot_$DATA.db

# backup 15 days snapshot file
if [ -d $BACKUP_DIR ]; then
   cd ${BACKUP_DIR} && find . -mtime +15 |xargs rm -f {} \;
fi
```

将脚本添加到cronjob中，定时每2小时执行一次

在Kubernetes中我们可以以cronjob的形式在集群中运行，并在pod中挂载外部存储，以做到数据的持久化

在创建Cronjob之前得把etcd certificates 通过secrets的形式挂载

```shell
D="$(mktemp -d)"
cp /etc/kubernetes/ssl/{ca,kubernetes,kubernetes-key}.pem $D
kubectl -n kube-system create secret generic etcd-certs --from-file="$D"
rm -fr "$D"
```

`etcd-backup-cronjob.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: etcd-backup-config
  namespace: kube-system
data:
  endpoints: "https://192.168.2.14:2379,https://192.168.2.15:2379,https://192.168.2.16:2379"

---
apiVersion: batch/v1beta1
kind: Cronjob
metadata:
  name: etcd_backup
  namespace: kube-system
spec:
  schedule: "*/30 * * * *"
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 3
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
			imagePullPolicy: IfNotPresent
			image: lwieske/etcdctl:v3.4.0
			env:
			- name: ENDPOINTS
			  valueFrom:
			    configMapKeyRef:
			      name: etcd-backup-config
			      key: endpoints
			command:
			- /bin/sh
			- -c
			- |
			  set -ex
              ETCDCTL_API=3 /usr/local/bin/etcdctl \
                --endpoints=$ENDPOINTS \
                --cacert=/certs/ca.pem \
                --cert=/certs/kubernetes.pem \
                --key=/certs/kubernetes-key.pem \
                snapshot save /data/backup/snapshot-$(date +"%Y%m%d").db
            volumeMounts:
            - name: etcd-backup
              mountPath: /data/backup
            - name: etcd-certs
              mountPath: /certs
          restartPolicy: OnFailure
          volumes:
          - name: etcd-backup
            persistentVolumeClaim:
              claimName: etcd-backup
          - name: etcd-certs
            secret:
              secretName: etcd-certs              
```

## etcd backup operator 

适用公有云环境Amazon S3存储

### Set up RBAC

Set up basic [RBAC rules](https://github.com/coreos/etcd-operator/blob/master/doc/user/rbac.md) for etcd operator:

```
$ example/rbac/create_role.sh
```

### Install etcd operator

Create a deployment for etcd operator:

```
$ kubectl create -f example/deployment.yaml
```

etcd operator will automatically create a Kubernetes Custom Resource Definition (CRD):

```
$ kubectl get customresourcedefinitions
NAME                                    KIND
etcdclusters.etcd.database.coreos.com   CustomResourceDefinition.v1beta1.apiextensions.k8s.io
```

### Uninstall etcd operator

Note that the etcd clusters managed by etcd operator will **NOT** be deleted even if the operator is uninstalled.

This is an intentional design to prevent accidental operator failure from killing all the etcd clusters.

To delete all clusters, delete all cluster CR objects before uninstalling the operator.

Clean up etcd operator:

```
kubectl delete -f example/deployment.yaml
kubectl delete endpoints etcd-operator
kubectl delete crd etcdclusters.etcd.database.coreos.com
kubectl delete clusterrole etcd-operator
kubectl delete clusterrolebinding etcd-operator
```

### Installation using Helm

etcd operator is available as a [Helm chart](https://github.com/kubernetes/charts/tree/master/stable/etcd-operator/). Follow the instructions on the chart to install etcd operator on clusters.[Alejandro Escobar](https://github.com/alejandroEsc) is the active maintainer.

## 恢复etcd集群

查看etcd集群信息

```shell
root@debian:~# export ETCDCTL_API=3
root@debian:~# etcdctl member list
610581aec980ff7, started, etcd1, https://192.168.2.14:2380, https://192.168.2.14:2379, false
d250eba9d73e634f, started, etcd2, https://192.168.2.15:2380, https://192.168.2.15:2379, false
de4be0d409c6cd2d, started, etcd3, https://192.168.2.16:2380, https://192.168.2.16:2379, false

```

停止Kubernetes集群中的kube-apiserver相关组件和etcd组件

```
systemctl stop kube-apiserver
systemctl stop kube-scheduler
systemctl stop kube-controller-manager
systemctl stop etcd
```

使用mv命令对etcd数据目录进行改名

```
mv /var/lib/etcd/data.etcd /var/lib/etcd/data.etcd_bak
```

分别在各个节点恢复数据，首先需要拷贝数据到每个etcd节点，假设备份数据存储在`/var/lib/etcd_backup/etcd_20190910133139.db`

```
scp /var/lib/etcd_backup/etcd_20190910133139.db root@etcd01:/var/lib/etcd_backup/
scp /var/lib/etcd_backup/etcd_20190910133139.db root@etcd02:/var/lib/etcd_backup/
scp /var/lib/etcd_backup/etcd_20190910133139.db root@etcd03:/var/lib/etcd_backup/
```

在需要恢复的所有etcd实例上执行恢复命令:

```
# ETCDCTL_API=3 etcdctl snapshot --ca-file=/etc/etcd/ssl/ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem  restore <备份数据> --name=<ETCD_NAME> --data-dir=<元数据存储路径> --initial-cluster=<ETCD_CLUSTER> --initial-cluster-token=<ETCD_INITIAL_CLUSTER_TOKEN>

```

同时启动etcd集群的所有etcd实例

```
# systemctl start etcd
```

检查etcd集群member及健康状态

```
# etcdctl --ca-file=/etc/etcd/ssl/ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem member list
# etcdctl --ca-file=/etc/etcd/ssl/ca.pem --cert-file=/etc/etcd/ssl/etcd.pem --key-file=/etc/etcd/ssl/etcd-key.pem cluster-health
```

启动master节点的所有kube-apiserver kube-scheduler kube-controller-manager服务:

```
# systemctl start kube-apiserver
systemctl start kube-scheduler
systemctl start kube-controller-manager
```