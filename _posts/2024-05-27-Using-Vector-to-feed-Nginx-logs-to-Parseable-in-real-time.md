---
layout: mypost
title:  Vector采集Nginx日志到Parseable做数据展示
categories: [Vector,Nginx,parseable]
---

## 背景
在之前的一篇文章中，我们接触到了Vector和Parseabel这2个日志组件
>[使用Tetragon在Kubernetes中进行eBPF日志分析](https://jaegerw2016.github.io/posts/2023/11/17/Get-started-with-ebpf-log-analytics-in-your-kubernetes-cluster.html)

众所周知，Nginx作为主流的Web服务器和反向代理服务器，存在最广泛的应用，这次就利用Nginx的Access.log的日志作为源，通过Vector这个Pipeline输送到Parseable做存储。

### 配置Nginx日志格式

使用标准 Nginx 访问日志，可以进一步使用自定义格式记录其他数据,具体可以查看Nginx文档,这里就不展开说明了。

```shell
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main ;
    server {
	...
	}
}
```
以上的配置，在Docker环境下Nginx容器通过`-v /var/log/nginx:/var/log/nginx`挂载宿主机路径,实现的accesss.log落盘持久化。后续Vector需要通过**File**的方式读取该路径下的log文件。

### Vector

`vector.yaml`配置文件
```yaml

api:
  enabled: true
  address: 0.0.0.0:8686
sources:
  nginx_logs:
    type: file
    include:
      - /var/log/nginx/access.log
transforms:
  process:
    type: remap
    inputs:
      - nginx_logs
    source: |-
      . = parse_regex!(.message, r'^(?P<ip>\d+\.\d+\.\d+\.\d+) - - \[(?P<date>\d+\/\w+\/\d+:\d+:\d+:\d+ \+\d+)\] +"(?P<http_request_methods>.+? )(?P<url>.+? )(?P<http_protocol>.+?)" (?P<http_status_code>\d+) (?P<body_bytes_sent>\d+) "(?P<http_referer>.+?)" "(?P<http_user_agent>.+?)" "(?P<http_x_forwarded_for>.+?)"')
      .timestamp = now()
      http_status_code =parse_int!(.http_status_code)
      if http_status_code >= 200 && http_status_code <= 299 {
         .status = "success"
      } else {
         .status = "error"
      }
sinks:
  parseable:
    type: http
    method: post
    batch:
      max_bytes: 10485760
      max_events: 1000
      timeout_secs: 10
    compression: gzip
    inputs:
      - process
    encoding:
      codec: json
    uri: 'http://10.100.10.100:8000/api/v1/ingest'
    auth:
      strategy: basic
      user: admin
      password: admin
    request:
      headers:
        X-P-Stream: vector
    healthcheck:
      enabled: true
      path: 'htpp://10.100.10.100:8000/api/v1/liveness'
      port: 8000

#sinks:
#  console:
#    inputs:
#      - process
#    target: stdout
#    type: console
#    encoding:
#      codec: json


```

vector.yaml的配置包含4块内容: **api**, **sources**, **transfer**, **sinks**


```
api:
  enabled: true
  address: 0.0.0.0:8686
```
**api** 负责vector是否提供api接口，和服务监听端口8686

```yaml
sources:
  nginx_logs:
    type: file
    include:
      - /var/log/nginx/access.log
```

**sources** 这块负责来自上游日志源的配置，这里是通过发**file**方式，读取nginx的特定路径下的access.log,其他的方式可以查看Vector官方文档

```yaml
transforms:
  process:
    type: remap
    inputs:
      - nginx_logs
    source: |-
      . = parse_regex!(.message, r'^(?P<ip>\d+\.\d+\.\d+\.\d+) - - \[(?P<date>\d+\/\w+\/\d+:\d+:\d+:\d+ \+\d+)\] +"(?P<http_request_methods>.+? )(?P<url>.+? )(?P<http_protocol>.+?)" (?P<http_status_code>\d+) (?P<body_bytes_sent>\d+) "(?P<http_referer>.+?)" "(?P<http_user_agent>.+?)" "(?P<http_x_forwarded_for>.+?)"')
      .timestamp = now()
      http_status_code =parse_int!(.http_status_code)
      if http_status_code >= 200 && http_status_code <= 299 {
         .status = "success"
      } else {
         .status = "error"
      }

```

**transforms** 负责对日志格式整形，也是vector这个组件我认为最灵活的地方，具体的Vector提供特定VRL语言，可以到Vector官方提供的[VRL Playground](https://playground.vrl.dev/) 测试正则表达式匹配,提取日志内容。

```yaml
sinks:
  parseable:
    type: http
    method: post
    batch:
      max_bytes: 10485760
      max_events: 1000
      timeout_secs: 10
    compression: gzip
    inputs:
      - process
    encoding:
      codec: json
    uri: 'http://10.100.10.100:8000/api/v1/ingest'
    auth:
      strategy: basic
      user: admin
      password: admin
    request:
      headers:
        X-P-Stream: vector
    healthcheck:
      enabled: true
      path: 'htpp://10.100.10.100:8000/api/v1/liveness'
      port: 8000

#sinks:
#  console:
#    inputs:
#      - process
#    target: stdout
#    type: console
#    encoding:
#      codec: json
```

**sinks** 负责将上面transforms的整形完的日志存储，这里以**http**方式，`POST`传给**parseable**专用log存储,日常在调试的过程中，通过**console**显示到标准输出。 
**X-P-Stream** 这个配置指定parseable的日志流名称，需要提前在Parseable里新建。

本次我们通过在Nginx容器旁安装Vector容器来实现Nginx的log日志传送，所以Vector容器同样需要挂载`-v /var/log/nginx:/var/log/nginx:ro` 来实现文件的读取
```
docker run --privileged --security-opt="seccomp=unconfined" -d   -v $PWD/vector.yaml:/etc/vector/vector.yaml:ro -v /var/log/nginx:/var/log/nginx:ro  -p 8686:8686   --name vector vector:0.38-debian
```

### Parseable
Parseable 是一个为现代云原生时代构建的日志分析平台。Parseable 使用无索引机制来组织和查询数据，实现低延迟和高吞吐量的数据摄取和查询。

同样在docker起一个Parseable容器

```shell

docker run -p 8000:8000 -v /opt/parseable/data:/parseable/data -v /opt/parseable/staging:/parseable/staging -e P_FS_DIR=/parseable/data -e P_STAGING_DIR=/parseable/staging parseable:latest parseable local-store

```
访问`http://{YOUR_HOST}/8000`,默认以`admin:admin`登录，具体用户名密码可以通过环境变量修改。

至此整个环境基本都已经完成，具体的微调就是一些log日志格式的字段排序展现。
