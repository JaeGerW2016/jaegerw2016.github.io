---
title: 构建基于alpine使用gnu libc的基础镜像
date: 2019-07-17 09:36:44
tags: docker
---
简单介绍一下Alpine版本中的musl libc和gnu libc的设定

### 背景

打开一个cli命令行运行一个alpine:3.9 容器

```shell
[root@localhost alpine-base]# docker run --name alpine --rm -it alpine:3.9 sh
/ #
```

重新打开一个CLI命令行，将`jdk-8u181-linux-x64.tar.gz` 复制进上面的alpine容器

```shell
docker cp jdk-8u181-linux-x64.tar.gz alpine:/tmp
```

```shell
[root@localhost alpine-base]# docker run --name alpine --rm -it alpine:3.9 sh
/tmp # ls
jdk-8u181-linux-x64.tar.gz
```

docker exec 进入容器解压`JDK 8u181`至/usr/local/share/java

```shell
/tmp # mkdir -p /usr/local/share/java
/tmp # 
/tmp # tar xzf jdk-8u181-linux-x64.tar.gz -C /usr/local/share/java
/tmp # cd /usr/local/share/java/
/usr/local/share/java # ls
jdk1.8.0_181
/usr/local/share/java # export PATH=$PATH:/usr/local/share/java/jdk1.8.0_181/bin
```

JDK只需要设定可执行目录至PATH搜索范围中即可，设定之后发现使用which可以找到java可执行文件，但是执行java -version却提示java not found

```shell
/usr/local/share/java # java -version
sh: java: not found
/usr/local/share/java #
/usr/local/share/java # which java
/usr/local/share/java/jdk1.8.0_181/bin/java
/usr/local/share/java # ldd /usr/local/share/java/jdk1.8.0_181/bin/java
	/lib64/ld-linux-x86-64.so.2 (0x7f53c1e69000)
	libpthread.so.0 => /lib64/ld-linux-x86-64.so.2 (0x7f53c1e69000)
	libjli.so => /usr/local/share/java/jdk1.8.0_181/bin/../lib/amd64/jli/libjli.so (0x7f53c1c53000)
	libdl.so.2 => /lib64/ld-linux-x86-64.so.2 (0x7f53c1e69000)
	libc.so.6 => /lib64/ld-linux-x86-64.so.2 (0x7f53c1e69000)
Error relocating /usr/local/share/java/jdk1.8.0_181/bin/../lib/amd64/jli/libjli.so: __rawmemchr: symbol not found
```

### 原因

ldd错误看起来好像是jli的的relocating提示错误信息，而实际上由于Alpine镜像使用的根本不是gnu libc而是musl libc，所以/lib64/ld-linux-x86-64.so.2是不存在的，而实际上/lib64都是不存在的。

```shell
/usr/local/share/java # ls /lib64
ls: /lib64: No such file or directory
```



gnu libc和musl libc号称兼容（部分兼容），基于缺什么补什么的原则，做个软链补上即可（stack overflow也有人有同样方法进行过验证）。由于Oracle的Java并不是完整的源码提供，所以Alpine也无法拿到源码去全编译来解决这个问题，大概这就是在Alpine镜像中更多的是OpenJDK的原因。

```shell
~ # mkdir /lib64
~ # ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2
~ # 
```

这个问题一旦得到对应，现象就不再诡异，再次执行java命令，中规中矩地提示了缺少的动态链接库

```shell
/usr/local/share/java # java -version
Error relocating /usr/local/share/java/jdk1.8.0_181/bin/../lib/amd64/jli/libjli.so: __rawmemchr: symbol not found
```

### [gnu libc和musl libc区别](https://wiki.musl-libc.org/functional-differences-from-glibc.html)

顺这个问题在Alpine的issue中很快找到了一个几年前的issue，写的很详细，jirutka 小哥还提到了Alpine的3S（Small/Simple/Secure）哲学，并坚持认为不应该在Alpine里面添加对于gnu libc的支持。

如果使用Oracle版本的JDK，或者类似需求，你应该去找支持glibc的linux发行版

### 解决方案

在Alpine里面安装glibc，让Alpine不再是Alpine

```dockerfile
FROM  alpine:3.9

MAINTAINER  jaeger "hzfyjgw2007@gmail.com" 

ENV LANG=C.UTF-8

# install GNU libc (aka glibc) C.UTF-8 locale and set timezone
RUN ALPINE_GLIBC_BASE_URL="https://github.com/sgerrand/alpine-pkg-glibc/releases/download" && \
    ALPINE_GLIBC_PACKAGE_VERSION="2.29-r0" && \
    ALPINE_GLIBC_BASE_PACKAGE_FILENAME="glibc-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    ALPINE_GLIBC_BIN_PACKAGE_FILENAME="glibc-bin-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    ALPINE_GLIBC_I18N_PACKAGE_FILENAME="glibc-i18n-$ALPINE_GLIBC_PACKAGE_VERSION.apk" && \
    apk add --no-cache --virtual=.build-dependencies wget ca-certificates && \
    wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub && \
    wget \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BASE_URL/$ALPINE_GLIBC_PACKAGE_VERSION/$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" && \
    apk add --no-cache \
        "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" && \
    \
    rm "/etc/apk/keys/sgerrand.rsa.pub" && \
    /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true && \
    echo "export LANG=$LANG" > /etc/profile.d/locale.sh && \
    \
    apk del glibc-i18n && \
    \
    rm "/root/.wget-hsts" && \
    apk del .build-dependencies && \
    rm \
        "$ALPINE_GLIBC_BASE_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_BIN_PACKAGE_FILENAME" \
        "$ALPINE_GLIBC_I18N_PACKAGE_FILENAME" && \
  echo 'http://mirrors.ustc.edu.cn/alpine/v3.9/main' > /etc/apk/repositories \
    && echo 'http://mirrors.ustc.edu.cn/alpine/v3.9/community' >>/etc/apk/repositories \
&& apk update && apk add tzdata \
&& ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \ 
&& echo "Asia/Shanghai" > /etc/timezone
```

构建alpine-base:3.9镜像

```shell
docker build -t alpine-base:3.9 .
```

运行基于alpine-base的容器并将JDK 8u181拷贝进容器

```
[root@localhost alpine-base]# docker run --name alpine-base --rm -it alpine-base:3.9 sh
/ # ls /tmp
jdk-8u181-linux-x64.tar.gz
/ # 
/ # mkdir -p /usr/local/share/java
/ # 
/ # cd /tmp
/tmp # 
/tmp # tar xzf jdk-8u181-linux-x64.tar.gz -C /usr/local/share/java
/tmp # export PATH=$PATH:/usr/local/share/java/jdk1.8.0_181/bin
/tmp # 
/tmp # java -version
java version "1.8.0_181"
Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)

```

### 总结

对镜像alpine有偏执的，可以通过这种方式来实现glibc的，当然直接使用支持glibc的linux发行版的镜像就没有这篇文章了，处处存在着妥协

### 参考文档
https://github.com/sgerrand/alpine-pkg-glibc
https://github.com/gliderlabs/docker-alpine/issues/11
http://mirror.cnop.net/jdk/linux/
