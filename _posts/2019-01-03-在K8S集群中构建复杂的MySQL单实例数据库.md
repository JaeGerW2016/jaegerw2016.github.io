---
title: 在K8S集群中构建复杂的MySQL单实例数据库
date: 2019-01-03 02:32:34
tags:
---
###文档说明
实验环境：kubernetes Version v1.9.6
网络CNI：fannel
存储CSI: NFS Dynamic Class

####前期准备
利用NFS动态提供Kubernetes后端存储卷[https://jimmysong.io/kubernetes-handbook/practice/using-nfs-for-persistent-storage.html]

###简单部署 （无法满足生产环境要求）
一、持久化存储卷pvc
`mysql-claim.yaml`

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "default" 
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
二、敏感数据Secret （最好base64编码加密）
`mysql-dev-secret.yaml`
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-dev-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: devadmin
  MYSQL_DATABASE: test

```
三、Deployment
`mysql-deployment.yaml`
```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mysql
  namespace: default
  labels:
    k8s-app: mysql
spec:
  replicas: 1
  selector: 
    matchLabels:
      k8s-app: mysql
  template:
    metadata:
      labels:
        k8s-app: mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        imagePullPolicy: Always
        envFrom:
        - secretRef:
            name: mysql-dev-secret
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: nfs-pv
          mountPath: /var/lib/mysql
      volumes:
        - name: nfs-pv
          persistentVolumeClaim:
            claimName: mysql-claim
```
四、Service
- 1无clusterIP  (kubernetes内部通过clusterIP访问)
```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    k8s-app: mysql
  clusterIP: None      
```
- 2 NodePort （通过NodePort暴露给集群外部访问，可以搭配外部LoadBalancer提供服务）

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    k8s-app: mysql
  ports:
  - nodePort: 33006
    protocol: TCP
    port: 3306
    targetPort: 3306
  type: NodePort
```
#####验证：
PVC  
![PVC.png](https://upload-images.jianshu.io/upload_images/3481257-a7c1c5027890aedb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Secret 
![Secret.png](https://upload-images.jianshu.io/upload_images/3481257-9f9ecfcd0af96b7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Deployment 
![Deployment.png](https://upload-images.jianshu.io/upload_images/3481257-1dfddb8d29ab7910.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Service
![Service.png](https://upload-images.jianshu.io/upload_images/3481257-1891d9ce71f53499.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###扩展部署（可用于生产环境）

一、持久化存储卷pvc (pv 详见前期准备nfs提供动态pv部分)
>tips: 确保存储卷后端的文件夹内为空，不然后期重新启动，会报错

`mysql-ex-claim.yaml`
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-ex-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "default" #根据实际情况调整
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi  #根据实际情况调整
```

二、敏感数据secret
`mysql-secret.yaml`
```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-user-pwd
data:
  mysql-root-pwd: aGFyZHRvZ3Vlc3M=
  mysql-app-user-pwd: aGFyZHRvZ3Vlc3M=
  mysql-test-user-pwd: aGFyZHRvZ3Vlc3M= 
```
![image.png](https://upload-images.jianshu.io/upload_images/3481257-5c45305ca4be3a28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


三、ConfigMap用户自定义配置
`mysql-cm.yaml` （mysql8.0配置）
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  custom.cnf: |
        [mysqld]
        default_storage_engine=innodb
        skip_external_locking
        lower_case_table_names=1
        skip_host_cache
        skip_name_resolve
```
`mysql-cm.yaml` （mysql5.7配置）
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  custom.cnf: |
        [mysqld]
        server-id=1
        log-bin
        expire_logs_days=7
        sync_binlog=0
        binlog_cache_size=1M

```
四、Deployment (包含初始化容器、健康检查)
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: mysql-ex
  name: mysql-ex
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-ex
  template:
     metadata:
       labels:
         app: mysql-ex
     spec:
       initContainers:
       - name: mysql-init
         image: busybox
         imagePullPolicy: IfNotPresent
         env:
         - name: MYSQL_TEST_USER_PASSWORD
           valueFrom:
             secretKeyRef:
               name: mysql-user-pwd
               key: mysql-test-user-pwd
         command:
           - sh
           - "-c"
           - |
             set -ex
             rm -rf /var/lib/mysql/lost+found
             cat > /docker-entrypoint-initdb.d/mysql-testdb-initt.sql <<EOF
             create database testdb default character set utf8;
             grant all on testdb.* to 'test'@'%' identified by '$MYSQL_TEST_USER_PASSWORD';
             flush privileges;
             EOF
             cat > /docker-entrypoint-initdb.d/mysql-appdb-init.sql <<EOF
             create table app(id int);
             insert into app values(1);
             commit;
             EOF
         volumeMounts:
         - name: mysql-data
           mountPath: /var/lib/mysql
         - name: mysql-initdb
           mountPath: /docker-entrypoint-initdb.d
       containers:
       #这边可以更改image: mysql:5.7
         - image: mysql 
           name: mysql
           imagePullPolicy: IfNotPresent
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-user-pwd
                 key: mysql-root-pwd
           - name: MYSQL_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysql-user-pwd
                 key: mysql-app-user-pwd
           - name: MYSQL_USER
             value: app
           - name: MYSQL_DATABASE
             value: appdb
           volumeMounts:
           - name: mysql-data
             mountPath: /var/lib/mysql
           - name: mysql-initdb
             mountPath: /docker-entrypoint-initdb.d
           - name: mysql-config
             mountPath: /etc/mysql/conf.d/
           ports:
           - name: mysql
             containerPort: 3306
           livenessProbe:
             exec:
               command:
               - /bin/sh
               - "-c"
               - MYSQL_PWD="${MYSQL_ROOT_PASSWORD}"
               - mysql -h 127.0.0.1 -u root -e "SELECT 1"
             initialDelaySeconds: 30
             timeoutSeconds: 5
             successThreshold: 1
             failureThreshold: 3
           readinessProbe:
             exec:
               command:
                 - /bin/sh
                 - "-c"
                 - MYSQL_PWD="${MYSQL_ROOT_PASSWORD}"
                 - mysql -h 127.0.0.1 -u root -e "SELECT 1"
             initialDelaySeconds: 10
             timeoutSeconds: 1
             successThreshold: 1
             failureThreshold: 3
       volumes:
         - name: mysql-data
           persistentVolumeClaim:
             claimName: mysql-ex-claim
         - name: mysql-initdb
           emptyDir: {}
         - name: mysql-config
           configMap:
             name: mysql-config
```

五、Service 
NodePort形式
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-ex
spec:
  selector:
    app: mysql-ex
  ports:
  - nodePort: 33016
    protocol: TCP
    port: 3306
    targetPort: 3306
  type: NodePort
```
> tips: 以上默认使用的image用的latest是mysql8.0 configmap有变化

验证:
![image.png](https://upload-images.jianshu.io/upload_images/3481257-32cbe2086e1ff8e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/3481257-6624af88ee028d4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


数据库连接情况：
![image.png](https://upload-images.jianshu.io/upload_images/3481257-2878646d0214156c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/3481257-2290ff07fd94aca9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



参考文档：
https://segmentfault.com/a/1190000014966962
