---
title: letsencrypt-auto自动续期报错
date: 2019-11-19 17:23
tag: cerbot
---

昨天使用**letsencrypt-auto**续期证书，输入`letsencrypt-auto renew`，没出现预期的“Congratulations, all renewals succeeded.”，而是意外的**“DNS problem: query timed out looking up CAA for jaeger.tk”**。咦，怎么突然就出错了呢？



```shell
root@vps:~# /opt/letsencrypt/letsencrypt-auto renew
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/jaeger.tk.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert is due for renewal, auto-renewing...
Plugins selected: Authenticator standalone, Installer None
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for jaeger.tk
http-01 challenge for www.jaeger.tk
Cleaning up challenges
Attempting to renew cert (jaeger.tk) from /etc/letsencrypt/renewal/jaeger.tk.conf produced an unexpected error: Problem binding to port 80: Could not bind to IPv4 or IPv6.. Skipping.
All renewal attempts failed. The following certs could not be renewed:
  /etc/letsencrypt/live/jaeger.tk/fullchain.pem (failure)

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

All renewal attempts failed. The following certs could not be renewed:
  /etc/letsencrypt/live/jaeger.tk/fullchain.pem (failure)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1 renew failure(s), 0 parse failure(s)

```

> 80 端口被占用 首先把nginx的服务停用 service nginx stop



```shell
root@vps:~# /opt/letsencrypt/letsencrypt-auto renew
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/jaeger.tk.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Cert is due for renewal, auto-renewing...
Plugins selected: Authenticator standalone, Installer None
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for jaeger.tk
http-01 challenge for www.jaeger.tk
Waiting for verification...
Challenge failed for domain www.jaeger.tk
http-01 challenge for www.jaeger.tk
Cleaning up challenges
Attempting to renew cert (jaeger.tk) from /etc/letsencrypt/renewal/jaeger.tk.conf produced an unexpected error: Some challenges have failed.. Skipping.
All renewal attempts failed. The following certs could not be renewed:
  /etc/letsencrypt/live/jaeger.tk/fullchain.pem (failure)

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

All renewal attempts failed. The following certs could not be renewed:
  /etc/letsencrypt/live/jaeger.tk/fullchain.pem (failure)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1 renew failure(s), 0 parse failure(s)

IMPORTANT NOTES:
 - The following errors were reported by the server:

   Domain: www.jaeger.tk
   Type:   dns
   Detail: DNS problem: query timed out looking up CAA for tk
```

上Google查原因和解决方案，发现不少人遇到同样的问题。在**Let’s Encrypt**的官方论坛和Server Fault，回帖基本上是针对个例的分析和解答，没看出问题原因和可行的解决方案。由于之前出现过域名校验失败导致certbot执行出错，过几小时再试就OK的例子，心想可能Let’s Encrypt服务器抽风了，明天也许就可以了。

首先要是弄清楚错误提示中的CAA是什么。CAA是证书颁发机构授权的缩写，域名的**CAA记录**用来指明能对域名签发证书的机构。该记录用来增强网站安全性，不是必须的。如果不存在，意味着所有的证书机构都可以给域名颁发证书。CAA记录的资源类型是type257，`dig`命令可以查看域名的CAA值，例如：`dig jaeger.tk type257`。

### 解决方法：

手动更新下CAA记录

```shell
dig jaeger.tk type257 

; <<>> DiG 9.9.5-9+deb8u11-Debian <<>> jaeger.tk type257
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49352
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;jaeger.tk.			IN	TYPE257

;; AUTHORITY SECTION:
jaeger.tk.		299	IN	SOA	ns01.freenom.com. soa.freenom.com. 1572417286 10800 3600 604800 3600

;; Query time: 305 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Tue Nov 19 08:45:53 UTC 2019
;; MSG SIZE  rcvd: 94
```

添加到定时任务中执行

```shell
0 0 1 * * dig jaeger.tk type257 >/dev/null 2>&1
0 0 1 * * /opt/letsencrypt/letsencrypt-auto renew >/dev/null 2>&1
```

