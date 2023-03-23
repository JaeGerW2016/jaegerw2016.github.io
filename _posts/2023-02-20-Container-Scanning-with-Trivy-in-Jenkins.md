---
layout:     post
title:     在 Jenkins 中使用 Trivy 进行容器扫描
subtitle:  Container Scanning with Trivy in Jenkins
date:       2023-02-20
author:     J
catalog: true
tags:
    - Jenkins
---
[Trivy](https://github.com/aquasecurity/trivy) 是个来自 Aqua Security的漏洞扫描系统，现已经被 Github Action、Harbor 等主流工具集成，能够非常方便的对镜像进行漏洞扫描，其扫描范围除了操作系统及其包管理系统安装的软件包之外，最近还加入了对 Ruby、PHP 等的漏洞检测，应该是该领域目前目前采用最广的开源工具之一了。
如果将扫描器集成到持续集成 (CI) 管道中，则可以在软件开发生命周期的早期识别并解决安全漏洞。
在这篇博文中，我们将研究Trivy。Trivy 是 AquaSecurity 团队的一款开源扫描器，可以扫描容器映像或文件系统路径以查找易受攻击的操作系统包或应用程序依赖项。
![](https://foreops.com/blog/trivy-intro/trivy-highlight.png)

## 安装Trivy

```
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.37.3
```
上面的过程可以通过以下脚本自动执行,按照路径`/usr/local/bin/trivy`

首先，让我们快速浏览一下 CLI

```
root@ubuntu:~# trivy --help
Scanner for vulnerabilities in container images, file systems, and Git repositories, as well as for configuration issues and hard-coded secrets

Usage:
  trivy [global flags] command [flags] target
  trivy [command]

Examples:
  # Scan a container image
  $ trivy image python:3.4-alpine

  # Scan a container image from a tar archive
  $ trivy image --input ruby-3.1.tar

  # Scan local filesystem
  $ trivy fs .

  # Run in server mode
  $ trivy server

Available Commands:
  aws         [EXPERIMENTAL] Scan AWS account
  config      Scan config files for misconfigurations
  filesystem  Scan local filesystem
  help        Help about any command
  image       Scan a container image
  kubernetes  [EXPERIMENTAL] Scan kubernetes cluster
  module      Manage modules
  plugin      Manage plugins
  repository  Scan a remote repository
  rootfs      Scan rootfs
  sbom        Scan SBOM for vulnerabilities
  server      Server mode
  version     Print the version
  vm          [EXPERIMENTAL] Scan a virtual machine image

Flags:
      --cache-dir string          cache directory (default "/root/.cache/trivy")
  -c, --config string             config path (default "trivy.yaml")
  -d, --debug                     debug mode
  -f, --format string             version format (json)
      --generate-default-config   write the default config to trivy-default.yaml
  -h, --help                      help for trivy
      --insecure                  allow insecure server connections when using TLS
  -q, --quiet                     suppress progress bar and log output
      --timeout duration          timeout (default 5m0s)
  -v, --version                   show version

Use "trivy [command] --help" for more information about a command.

```
让我们试一试trivy，看看它的实际效果。

`trivy image alpine:3.12`
```
root@ubuntu:~# trivy image alpine:3.12
2023-02-20T17:21:42.206+0800    INFO    Need to update DB
2023-02-20T17:21:42.207+0800    INFO    DB Repository: ghcr.io/aquasecurity/trivy-db
2023-02-20T17:21:42.207+0800    INFO    Downloading DB...
35.71 MiB / 35.71 MiB [---------------------------------------------------------------------------------------------------------------------------------] 100.00% 3.74 MiB p/s 9.8s
2023-02-20T17:21:54.539+0800    INFO    Vulnerability scanning is enabled
2023-02-20T17:21:54.540+0800    INFO    Secret scanning is enabled
2023-02-20T17:21:54.541+0800    INFO    If your scanning is slow, please try '--security-checks vuln' to disable secret scanning
2023-02-20T17:21:54.542+0800    INFO    Please see also https://aquasecurity.github.io/trivy/v0.36/docs/secret/scanning/#recommendation for faster secret detection
2023-02-20T17:21:56.710+0800    INFO    Detected OS: alpine
2023-02-20T17:21:56.710+0800    INFO    Detecting Alpine vulnerabilities...
2023-02-20T17:21:56.770+0800    INFO    Number of language-specific files: 0
2023-02-20T17:21:56.772+0800    WARN    This OS version is no longer supported by the distribution: alpine 3.12.12
2023-02-20T17:21:56.772+0800    WARN    The vulnerability detection may be insufficient because security updates are not provided

alpine:3.12 (alpine 3.12.12)

Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 1)

┌─────────┬────────────────┬──────────┬───────────────────┬───────────────┬─────────────────────────────────────────────────────────────┐
│ Library │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │                            Title                            │
├─────────┼────────────────┼──────────┼───────────────────┼───────────────┼─────────────────────────────────────────────────────────────┤
│ zlib    │ CVE-2022-37434 │ CRITICAL │ 1.2.12-r0         │ 1.2.12-r2     │ zlib: heap-based buffer over-read and overflow in inflate() │
│         │                │          │                   │               │ in inflate.c via a...                                       │
│         │                │          │                   │               │ https://avd.aquasec.com/nvd/cve-2022-37434                  │
└─────────┴────────────────┴──────────┴───────────────────┴───────────────┴─────────────────────────────────────────────────────────────┘

```
可以看出`apline:3.12`存在一个`CRITICAL`级别的`Vulnerability:CVE-2022-37434` 显示`zlib`的依赖存在高危漏洞风险

## Jenkins CI集成

`jenkinsfile`

```groovy
pipeline {
    agent any
    stages {
        stage('Pull Dcker image') {
            steps {
                echo 'Pull image from registry'
                sh 'whoami'
                sh 'docker pull ${DOCKER_IMAGE}:${VERSION}'
            }
        }
        stage('Scan') {
            steps {
                // Install trivy
                // sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.37.3'
                // sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl'
                sh 'curl -sfL --proxy http://192.168.2.122:7890 https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.37.3'
                sh 'curl -sfL --proxy http://192.168.2.122:7890 https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl'


                // Scan all vuln levels
                sh 'mkdir -p reports'
                sh 'trivy image --format template --template "@html.tpl" -o reports/CVE_report.html ${DOCKER_IMAGE}:${VERSION}'
                publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'CVE_report.html',
                    reportName: 'Trivy Scan',
                    reportTitles: 'Trivy Scan'
                ]

                // Scan again and fail on CRITICAL vulns
                sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH ${DOCKER_IMAGE}:${VERSION}'
            }
        }
    }
}

```

## 验证

[![-2023-02-21-16-05-540f3ef0ee57466ba7.md.png](https://youjb.com/images/2023/02/21/-2023-02-21-16-05-540f3ef0ee57466ba7.md.png)](https://youjb.com/image/ceV)
[![-2023-02-21-16-06-29eee8f2d8fe878e79.md.png](https://youjb.com/images/2023/02/21/-2023-02-21-16-06-29eee8f2d8fe878e79.md.png)](https://youjb.com/image/ceL)
[![-2023-02-21-16-08-13b117b50cfd195252.md.png](https://youjb.com/images/2023/02/21/-2023-02-21-16-08-13b117b50cfd195252.md.png)](https://youjb.com/image/ceq)
