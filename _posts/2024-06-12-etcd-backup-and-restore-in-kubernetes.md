---
layout: mypost
title: ETCD Backup and Restore in Kubernetes
categories: [ETCD,Kubernetes]
---

**在这篇博客中，我们将讨论 Kubernetes 中 ETCD 备份和恢复的重要性，并使用 Kubernetes 文档作为指南引导您完成整个过程。**


## 为什么备份ETCD至关重要

ETCD是Kubernetes使用的分布式键值存储，用于存储所有集群数据，包括配置信息，集群服务状态。备份ETCD数据重要的原因如下：

- 灾难恢复: 如果发生硬件故障、数据损坏或其他不可预见的事件，备份ETCD可确保可以将Kubernetes集群恢复到以前的状态。
- 升级和迁移: 在升级或迁移集群之前，备份ETCD至关重要。如果在此过程中出现问题，可以通过数据备份回复到以前的状态。
- 数据丢失预防: 防止因人为错误或配置错误而导致数据丢失。

## 如何使用etcdctl备份etcd

### 准备环境

- 1. Kubernetes集群
- 2. etcdctl CLI

etcd的常见的运行方式有2种：静态Pod容器化部署和独立部署的方式

### 独立部署方式
这是最基本的运行方式，在服务器或虚拟机上直接运行 ETCD 实例。可以使用二进制文件、系统包管理器（如 apt、yum）或从源码编译安装 ETCD。

可以使用以下操作安装 ETCDCTL
```
sudo apt update
sudo apt install etcd-client

or
ETCD_VER =v3.5.0 
GOOGLE_URL =https://storage.googleapis.com/etcd 
DOWNLOAD_URL = ${GOOGLE_URL}

curl -L ${DOWNLOAD_URL}/v${ETCD_VER}/etcd-v${ETCD_VER}-linux-amd64.tar.gz -o etcd-v${ETCD_VER}-linux-amd64.tar.gz 
tar xzvf etcd-v${ETCD_VER}-linux-amd64.tar.gz 
cd etcd-v${ETCD_VER}-linux-amd64 
sudo mv etcdctl /usr/local/bin/
```

### 收集etcd的环境信息

要获取 etcd 快照，您需要以下信息：

- etcd endpoints (default https://127.0.0.1:2379)
- ca certificate file
- Server certificate file
- Server key file

一般etcd的配置文件在`/etc/etcd/etcd.conf`这个路径下面，或者以`/etc/etcd.env` 定义环境变量的方式
这里就在etcd独立部署的服务器上以`/etc/etcd.env`的方式来获取以上关键信息

```
...

# CLI settings
ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
ETCDCTL_CACERT=/etc/ssl/etcd/ssl/ca.pem
ETCDCTL_KEY=/etc/ssl/etcd/ssl/admin-node1-key.pem
ETCDCTL_CERT=/etc/ssl/etcd/ssl/admin-node1.pem

```

### 获取ETCD快照

```
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/ssl/etcd/ssl/ca.pem \
--cert=/etc/ssl/etcd/ssl/admin-node1.pem \
--key=/etc/ssl/etcd/ssl/admin-node1.pem \
snapshot save /opt/etcd-data-backup/snapshot.db
```
通过以后的命令来创建快照备份，成功执行之后，会在`/opt/etcd-data-backup`保存一份`snapshot.db`的快照文件。

### 验证Snapshot快照

```
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/etcd-data-backup/snapshot.db
Deprecated: Use `etcdutl snapshot status` instead.
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a3b08af8 |    35743 |       1842 |     7.0 MB |
+----------+----------+------------+------------+
```
以上信息包含哈希、修订版本、key键值数、快照大小。

### 从快照恢复ETCD

```
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-data-backup/snapshot.db
```

使用特定的数据路径恢复，可以指定`--data-dir /var/lib/etcd`,可以控制恢复过程中etcd的数据存储位置。


### 创建Crontab 计划任务

```
# m h  dom mon dow   command
# Run daily at 2 AM
0 2 * * *  ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/admin-node1.pem --key=/etc/ssl/etcd/ssl/admin-node1-key.pem snapshot save /opt/etcd-data-backup/snapshot.db
```

### 静态Pod部署方式

确定ETCD的Pod所在Node节点,打上label标签，通过selector选择，让CronJob运行在etcd的pod所在的Node

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Run daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5.0
            command:
            - /bin/sh
            - -c
            - |
              export ETCDCTL_API=3
              etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/certs/ca.crt --cert=/certs/etcd.crt --key=/certs/etcd.key snapshot save /backups/snapshot.db
            volumeMounts:
            - name: etcd-certs
              mountPath: /certs
              readOnly: true
            - name: etcd-backup
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            secret:
              secretName: etcd-certs
          - name: etcd-backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
```

```
kubectl apply -f etcd-backuo-cronjob.yaml
```

```
apiVersion: batch/v1
kind: Job
metadata:
  name: etcd-restore
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: etcd-restore
        image: bitnami/etcd:3.5.0
        command:
        - /bin/sh
        - -c
        - |
          export ETCDCTL_API=3
          etcdctl snapshot restore /backups/snapshot.db --data-dir=/var/lib/etcd/new.etcd
        volumeMounts:
        - name: etcd-backup
          mountPath: /backups
        - name: etcd-data
          mountPath: /var/lib/etcd/new.etcd
      restartPolicy: OnFailure
      volumes:
      - name: etcd-backup
        persistentVolumeClaim:
          claimName: etcd-backup-pvc
      - name: etcd-data
        persistentVolumeClaim:
          claimName: etcd-data-pvc
```
```
kubectl apply -f etcd-restore-job.yaml
```

