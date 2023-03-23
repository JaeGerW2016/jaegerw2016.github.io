---
title: 如何在Docker容器中动态设置系统时间
date: 2019-07-29 16:23
tag: docker
---

#### 如何在Docker容器中动态设置系统时间

### 解决方法：

在容器中伪造它 [libfaketime](https://github.com/wolfcw/libfaketime)拦截所有系统调用程序用于检索当前时间和日期.

`demo`

```dockerfile
From debian:stretch as builder

RUN apt-get update && apt-get install -y --no-install-recommends git ca-certificates build-essential 
WORKDIR /
RUN git clone https://github.com/wolfcw/libfaketime.git
WORKDIR /libfaketime/src
RUN make && make install

FROM python:3.6-slim-stretch

COPY --from=builder /usr/local/lib/faketime/libfaketime.so.1 /usr/local/lib/faketime/libfaketime.so.1

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo "Asia/Shanghai" > /etc/timezone

COPY faketime.py entrypoint.sh /

ENTRYPOINT ["/entrypoint.sh"]

```

`entrypoint.sh`

```shell
#!/bin/sh

export LD_PRELOAD=/usr/local/lib/faketime/libfaketime.so.1
export FAKETIME_NO_CACHE=1

python /faketime.py
```

`faketime.py`

```python
#!/usr/bin/env python

import os
from datetime import datetime

def get_real_time():
    print(datetime.today())

def set_time_with_absolute_dates():
    os.environ["FAKETIME"] = "2020-12-24 20:30:00"
    print(datetime.today())

def set_time_with_start_at_dates():
    os.environ["FAKETIME"] = "@2020-12-24 20:30:00"
    print(datetime.today())

def set_time_with_day_offset():
    os.environ["FAKETIME"] = "+15d"
    print(datetime.today())

def set_time_with_hour_offset():
    os.environ["FAKETIME"] = "+1.5h"
    print(datetime.today())
    
def set_time_with_min_offset():
    os.environ["FAKETIME"] = "-10m"
    print(datetime.today())

def set_time_with_second_offset():
    #set the faked time 2 minutes (120 seconds) behind the real time
    os.environ["FAKETIME"] = "-120"
    print(datetime.today())

if __name__ == "__main__":
    get_real_time()
    set_time_with_absolute_dates()
    set_time_with_start_at_dates()
    set_time_with_day_offset()
    set_time_with_hour_offset()
    set_time_with_min_offset()
    set_time_with_second_offset()
```



#### 验证结果：

```shell
[root@master faketime]# docker run -it 31431596/stretch-faketime:v0.1
2019-07-29 16:23:21.064940
2020-12-24 20:30:00
2020-12-24 20:30:00.000033
2019-08-13 16:23:21.065352
2019-07-29 17:53:21.065438
2019-07-29 16:13:21.065509
2019-07-29 16:21:21.065577

```

