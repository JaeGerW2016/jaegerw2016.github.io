---
layout:     post
title:     Openssl 在Ubuntu 22.04上升级报错
subtitle:  An Issue with OpenSSL Upgrade on Ubuntu 22.04
date:       2022-11-10
author:     J
catalog: true
tags:
    - Linux
---


## 背景

OpenSSL 已发布 3.0.7 修复两个高危漏洞：CVE-2022-3786 和 CVE-2022-3602。官方建议 OpenSSL 3.0.x 用户应升级到 OpenSSL 3.0.7，因为这两个漏洞影响 OpenSSL 3.0.0 至 3.0.6 版本，不影响 OpenSSL 1.1.1 和 1.0.2。

于是我在查看[Ubuntu CVEs Reports](https://ubuntu.com/security/cves)


- [CVE-2022-3786](https://ubuntu.com/security/CVE-2022-3786)
- [CVE-2022-3602](https://ubuntu.com/security/CVE-2022-3602)


可以在报告中看出Ubuntu社区通过更新Openssl(3.0.2-0ubuntu1.7)修复上述漏洞

[![-2022-11-10-11-34-2579a73f6d1bce61fd.md.png](https://youjb.com/images/2022/11/10/-2022-11-10-11-34-2579a73f6d1bce61fd.md.png)](https://youjb.com/image/cKd)


然而在Ubuntu22.04 上进行正常的Curl 命令操作时出现异常

```
user@localhost:~$ sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0
curl: (35) error:0A000126:SSL routines::unexpected eof while reading

```

## Stackoverflow搜索一把之后
## 解决方案如下：


```
Upgrade your nginx 1.18.0 to the mainline version and the problem will be fixed. To do so:


Execute as sudo: curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor >/usr/share/keyrings/nginx-signing.gpg

ADD THE LINES in /etc/apt/sources.list:

deb [signed-by=/usr/share/keyrings/nginx-signing.gpg] https://nginx.org/packages/mainline/ubuntu/ jammy nginx
deb-src [signed-by=/usr/share/keyrings/nginx-signing.gpg] https://nginx.org/packages/mainline/ubuntu/ jammy nginx

sudo apt update
sudo apt install nginx


```

## 参考链接
https://www.linux.org/threads/curl-error-35-ssl-connect-error.41639/
https://stackoverflow.com/questions/72627218/openssl-error-messages-error0a000126ssl-routinesunexpected-eof-while-readin
