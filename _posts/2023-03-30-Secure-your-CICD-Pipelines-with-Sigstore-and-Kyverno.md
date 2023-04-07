---
layout: mypost
title: 使用 Sigstore 和 Kyverno 来签名和验证图像保护CI/CD Pipeline
categories: [Sigstore, Kyverno]
---
## 背景
随着云原生和容器应用交付的兴起，我们如何通过确保我们在 Kubernetes 集群上运行的容器镜像是可信的,来提高软件在软件开发周期中的供应链的安全性？我们可以通过给容器镜像添加数字签名来做到这一点。

对容器镜像进行添加数字签名可确保我们的软件供应链安全性得到提高，并且没有人可以通过CICD将带有漏洞或者恶意代码的镜像部署到我们的 Kubernetes 集群中。

## Sigstore 和 Kyverno
[Sigstore](https://www.sigstore.dev/) 提供了一种以公开、透明和可访问的方式更好地保护软件供应链的方法。
`Sigstore` 提供了一种使用数字签名来签署、验证和保护软件的方法。有了这个，您可以将软件追溯到其源头，从而确信它处于当前状态并且没有以任何方式被更改。

`Sigstore` 提供了一种生成数字签名和验证工件所需的密钥对的方法。`Sigstore` 为您提供透明分类帐技术的好处，这意味着任何人都可以找到并验证签名并检查是否有人更改了源代码、工件或您的构建平台。

对镜像数字签名后，我们需要确保只有具有有效数字签名并经过验证的容器镜像才能部署到我们的Kubernetes集群。这个时候就需要`Kyverno`这个云原生策略引擎来支持。
[Kyverno](https://kyverno.io/docs/introduction/) 作为 Kubernetes 策略资源进行管理。您不需要一种新的语言来编写策略。
`Kyverno` 在 Kubernetes 集群中作为动态准入控制器运行。它从 kube-apiserver 接收验证和改变准入 webhook HTTP 回调，并应用匹配策略以返回执行准入策略或拒绝请求的结果。

`Kyverno` 策略可以验证、改变和生成 Kubernetes 资源。您可以使用 kubectl、kustomize 和 git 等工具来管理 `Kyverno` 策略。它有一个 CLI，可用于测试策略和验证资源作为 CI/CD 管道的一部分。

`Kyverno` 依靠来自[Sigstore](https://www.sigstore.dev/)项目的[Cosign](https://github.com/sigstore/cosign)来验证图像。

### 先决条件
- Kubernetes Cluster
- kubectl
- Cosign CLI
- Kyverno Install In Kubernetes Cluster
- Docker or Nerdctl Build Image

### Cosign CLI安装

```
#With the Cosign binary or rpm/dpkg package

# binary
wget "https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-linux-amd64"
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

# rpm
wget "https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign-2.0.0.x86_64.rpm"
rpm -ivh cosign-2.0.0.x86_64.rpm

# dkpg
wget "https://github.com/sigstore/cosign/releases/download/v2.0.0/cosign_2.0.0_amd64.deb"
dpkg -i cosign_2.0.0_amd64.deb

```


### Cosign 生成密钥对

```
root@ubuntu:/opt/sigstore-kyverno# cosign generate-key-pair
root@ubuntu:/opt/sigstore-kyverno# ls
cosign_2.0.0_amd64.deb  cosign.key  cosign.pub  Dockerfile  index.html
```

我们生成一个全新的密钥对，为`cosign.key`和`cosign.pub`两个文件


### Docker 构建demo镜像

```
root@ubuntu:/opt/sigstore-kyverno# docker build -t 314315960/nginx-demo:unsigned .
Sending build context to Docker daemon   43.7MB
Step 1/3 : FROM nginx:latest
 ---> 080ed0ed8312
Step 2/3 : LABEL maintainer="Joe.Devops Unsigned"
 ---> Running in 08ef42fad9cc
Removing intermediate container 08ef42fad9cc
 ---> ec14fc0eb9f9
Step 3/3 : COPY index.html /usr/share/nginx/html
 ---> 854637f8e58a
Successfully built 854637f8e58a
Successfully tagged 314315960/nginx-demo:unsigned
root@ubuntu:/opt/sigstore-kyverno# docker push 314315960/nginx-demo:unsigned
The push refers to repository [docker.io/314315960/nginx-demo]
6a2579e41fe0: Pushed
ff4557f62768: Layer already exists
4d0bf5b5e17b: Layer already exists
95457f8a16fd: Layer already exists
a0b795906dc1: Layer already exists
af29ec691175: Layer already exists
3af14c9a24c9: Layer already exists
unsigned: digest: sha256:41a0becd866388eccdf17ec434899f2423d93c2e61902ea9b58475260e84e1e3 size: 1777
```
### Cosign验证未添加数字签名的图像

```
root@ubuntu:/opt/sigstore-kyverno# cosign verify --key cosign.pub 314315960/nginx-demo:unsigned | jq -r .
Error: no matching signatures:

main.go:69: error during command execution: no matching signatures:

```

### Cosign添加数字签名

```
root@ubuntu:/opt/sigstore-kyverno# cosign sign --key cosign.key 314315960/nginx-demo:signed
Enter password for private key:
WARNING: Image reference 314315960/nginx-demo:signed uses a tag, not a digest, to identify the image to sign.
    This can lead you to sign a different image than the intended one. Please use a
    digest (example.com/ubuntu@sha256:abc123...) rather than tag
    (example.com/ubuntu:latest) for the input to cosign. The ability to refer to
    images by tag will be removed in a future release.


        Note that there may be personally identifiable information associated with this signed artifact.
        This may include the email address associated with the account with which you authenticate.
        This information will be used for signing this artifact and will be stored in public transparency logs and cannot be removed later.

By typing 'y', you attest that you grant (or have permission to grant) and agree to have this information stored permanently in transparency logs.
Are you sure you would like to continue? [y/N] y
tlog entry created with index: 17353368
Pushing signature to: index.docker.io/314315960/nginx-demo

```

我们可以在Docker的镜像仓库看到`Cosign`自动将一个带有`tag::sha256-467287fe0f76359cc278356089839b5858469ef87d3a88212f85d52e8d985306.sig`的镜像推送仓库

![-2023-04-07-16-16-033a98f1ffe4dabef1.png](https://youjb.com/images/2023/04/07/-2023-04-07-16-16-033a98f1ffe4dabef1.png)

### Cosign验证镜像

```shell
root@ubuntu:/opt/sigstore-kyverno# cosign verify --key cosign.pub 314315960/nginx-demo:signed | jq -r .

Verification for index.docker.io/314315960/nginx-demo:signed --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
[
  {
    "critical": {
      "identity": {
        "docker-reference": "index.docker.io/314315960/nginx-demo"
      },
      "image": {
        "docker-manifest-digest": "sha256:467287fe0f76359cc278356089839b5858469ef87d3a88212f85d52e8d985306"
      },
      "type": "cosign container image signature"
    },
    "optional": {
      "Bundle": {
        "SignedEntryTimestamp": "MEYCIQDQMwh8x6o1IbZrcyYW71fGKudtHBMWGTRIwvb5NNbmsgIhALcLxSlunjGmRCvPihUpNg3X17DSDa61PpfdIaYHt00E",
        "Payload": {
          "body": "eyJhcGlWZXJzaW9uIjoiMC4wLjEiLCJraW5kIjoiaGFzaGVkcmVrb3JkIiwic3BlYyI6eyJkYXRhIjp7Imhhc2giOnsiYWxnb3JpdGhtIjoic2hhMjU2IiwidmFsdWUiOiIyYmZmYzA5YjI5ZmZjOTE5OTlkMGQxMGJmODBhMDRiZTU4YmY4NGJkYzNlNzQzOTIyOTc1YTIzNmJlYTczOTZmIn19LCJzaWduYXR1cmUiOnsiY29udGVudCI6Ik1FVUNJQ1d1TGtwcWZFYUVDMUloZkpIY0pDRGN0RkhxQzR0U2UyWUt6ZXc2dzEwMkFpRUEreGY0dUNkYmNIblVKYkxGK2dYRXpvb0svSzBMN3BrWWxxaDBoNTR2dU1BPSIsInB1YmxpY0tleSI6eyJjb250ZW50IjoiTFMwdExTMUNSVWRKVGlCUVZVSk1TVU1nUzBWWkxTMHRMUzBLVFVacmQwVjNXVWhMYjFwSmVtb3dRMEZSV1VsTGIxcEplbW93UkVGUlkwUlJaMEZGVG5aU1RTOHlUVkkyVkdoQ1YwaHBUV1p0S3pac2VUZDRNVE5STXdwUmMySkxka2x0WlVwb05XRm1kVkZLYkVKb1JIbEdWRVowYm5CR1pIbEtTMnh0UzAxblNuVXlXakZ3UW5obVVUVkNORWwyVkZac05YcEJQVDBLTFMwdExTMUZUa1FnVUZWQ1RFbERJRXRGV1MwdExTMHRDZz09In19fX0=",
          "integratedTime": 1680855335,
          "logIndex": 17353368,
          "logID": "c0d23d6ad406973f9559f3ba2d1ca01f84147d8ffc5b8445c224f98b9591801d"
        }
      }
    }
  }
]

```

### 在Kubernetes 中安装 Kyverno

```
#Add Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

#Install Kyverno using Helm
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace

#Check that Kyverno pods are in running state
root@node1:/opt/sigstore-kyverno# kubectl get pod -n kyverno
NAME                                         READY   STATUS    RESTARTS   AGE
kyverno-696d66bc7b-k5frw                     1/1     Running   0          92m
kyverno-cleanup-controller-5dc5d7774-frhz6   1/1     Running   0          92m

```



### Kyverno策略

`check-image-sign-policy.yaml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: check-image
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "314315960/nginx-demo*"
        attestors:
        - count: 1
          entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAENvRM/2MR6ThBWHiMfm+6ly7x13Q3
                QsbKvImeJh5afuQJlBhDyFTFtnpFdyJKlmKMgJu2Z1pBxfQ5B4IvTVl5zA==
                -----END PUBLIC KEY-----
```

```
root@node1:/opt/sigstore-kyverno# kubectl apply -f check-image-sign-policy.yaml
clusterpolicy.kyverno.io/check-image created
root@node1:/opt/sigstore-kyverno#
root@node1:/opt/sigstore-kyverno#
root@node1:/opt/sigstore-kyverno# kubectl get clusterpolicy
NAME          BACKGROUND   VALIDATE ACTION   READY   AGE
check-image   false        Enforce           true    21s
```
默认情况下，此条策略的`validationFailureAction`为强制并且`failurePolicy`设置为`Fail`。
`verifyImages`为Kyverno 中 ClusterPolicy 的规则检查所有匹配的容器镜像签名。
此次本文中我们正在验证`314315960/nginx-demo`这个镜像所有生成的标签，因此使用`通配符*`


### 验证Pod运行
接下来让我们通过部署一个 pod 来检查验证过程，该 pod 运行我们从 DockerHub 签名的图像。
`myunsignedapp.yaml`

```
apiVersion: v1
kind: Pod
metadata:
 name: myunsignedapp
spec:
 containers:
 - name: app
   image: 314315960/nginx-demo:unsigned
   imagePullPolicy: Always
```

`mysignedapp.yaml`

```
apiVersion: v1
kind: Pod
metadata:
 name: mysignedapp
spec:
 containers:
 - name: app
   image: 314315960/nginx-demo:signed
   imagePullPolicy: Always

```

### 尝试运行未签名的图像将产生如下策略错误
```
root@node1:/opt/sigstore-kyverno# kubectl apply -f mysignedapp.yaml
pod/mysignedapp created

root@node1:/opt/sigstore-kyverno# kubectl apply -f myunsignedapp.yaml
Error from server: admission webhook "mutate.kyverno.svc" denied the request:

resource Pod/default/unsigned was blocked due to the following policies

check-image:
  check-image: 'image verification failed for 314315960/nginx-demo:unsigned:
    signature not found'
```
