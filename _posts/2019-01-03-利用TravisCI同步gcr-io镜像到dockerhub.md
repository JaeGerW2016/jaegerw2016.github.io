---
title: 利用TravisCI同步gcr.io镜像到dockerhub
date: 2019-01-03 02:53:08
tags:
---

前期准备
一、背景介绍
         由于国内网络原因，gcr.io 仓库里的镜像是无法直接拉取到的，这给开发工作造成了极大的不便

本文介绍一种方法能够实现自动化地定期地将 gcr.io 仓库中的镜像同步到个人 DockerHub 账户

>实现该方案需要满足以下条件：

-   已注册 GitHub 账号 https://github.com/

-  已注册 DockerHub 账号 https://hub.docker.com

-  已注册 Google Cloud 账号 https://cloud.google.com/

-  已注册 Travis CI 账号 https://www.travis-ci.org/


二、实现步骤

一、Google Cloud 服务账户，以读取 gcr.io 仓库中的镜像列表 `生成json文件 保存文件备用`
登录 Google Cloud 控制台，点击菜单 “IAM和管理” -> “服务账号”
![image.png](https://upload-images.jianshu.io/upload_images/3481257-3bd9e6bde00d9b4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

填入 “账号名称” 和 “账号ID”，点击 “创建”，之后授予权限，角色选择 “容器分析备注查看者”，点击 “继续”
![image.png](https://upload-images.jianshu.io/upload_images/3481257-22c7e14b7535e303.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 点击 “创建密钥”
![image.png](https://upload-images.jianshu.io/upload_images/3481257-75f86a1303ba9330.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

推荐JSON格式
![image.png](https://upload-images.jianshu.io/upload_images/3481257-83f2498167615271.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



二、DockerHub 登录密钥文件 `保存文件备用`
>DockerHub Login 登录，在家目录.docker目录下生成config.json

![Docker login](https://upload-images.jianshu.io/upload_images/3481257-cc7904186836d5ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Docker](https://upload-images.jianshu.io/upload_images/3481257-b64aafba67c47e84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

三、创建 SSH 密钥，并在 GitHub 上进行授权 `id_rsa文件保存备用`
这部分省略，自行查询

四、创建 GitHub 项目，配置 Travis CI 策略

GitHub 项目名称 https://github.com/JaeGerW2016/mirrorgcrio
 ```
# 拉取github项目到本地
git clone https://github.com/JaeGerW2016/mirrorgcrio.git

cd mirrorgcrio
touch .travis.yml
#将之前生成的 gcloud.config.json、config.json、id_rsa 放置到项目根目录

tar czvf config.tar.gz gcloud.config.json config.json id_rsa

#docker 拉取一个travis-ci镜像生成加密文件
docker run -it -v /opt/travis-ci/mirrorgcrio:/root shidaqiu/travis-cli

travis login
#登录成功后，在容器执行操作
travis encrypt-file config.tar.gz --add

#退出容器，由于/opt/travis-ci/mirrorgcrio挂载到容器做持久化，会在根目录下生成config.tar.gz.enc
mv config.tar.gz.enc .travis
#防止后期git push到github上
rm -f config.tar.gz gcloud.config.json config.json id_rsa
````

编辑`.travis.yml`
```
sudo: required
language: python
python:
- '2.7'
addons:
  apt:
    packages:
    - docker-ce
branches:
  only:
  - master
install:
- git remote -v
script:
- "./get_image.sh"
before_install:
- export start_time=$(date +%s)
- mkdir -p ~/.docker
- mkdir -p ~/.ssh
#替换自己的加密文件
- openssl aes-256-cbc -K $encrypted_1fc90f464345_key -iv $encrypted_1fc90f464345_iv
  -in .travis/config.tar.gz.enc -out ~/config.tar.gz -d
- tar xf ~/config.tar.gz -C ~
- mv ~/id_rsa ~/.ssh/id_rsa
- mv ~/config.json ~/.docker/config.json
- chmod 600 ~/.ssh/id_rsa
- chmod 600 ~/.docker/config.json
- eval $(ssh-agent)
- ssh-add ~/.ssh/id_rsa

```

编辑`get_image.sh`

```
#!/bin/bash

GCR_NAMESPACE=gcr.io/google-containers
#自行替换自己的Dockerhub
DOCKERHUB_NAMESPACE=mirrorgcrio

today(){
   date +%F
}

git_init(){
    git config --global user.name "xxxx"
    git config --global user.email xxxx@qq.com
    git remote rm origin
    git remote add origin git@github.com:JaeGerW2016/mirrorgcrio.git
    git pull
    if git branch -a |grep 'origin/develop' &> /dev/null ;then
        git checkout develop
        git pull origin develop
        git branch --set-upstream-to=origin/develop develop
    else
        git checkout -b develop
        git pull origin develop
    fi
}

git_commit(){
     local COMMIT_FILES_COUNT=$(git status -s|wc -l)
     local TODAY=$(today)
     if [ $COMMIT_FILES_COUNT -ne 0 ];then
        git add -A
        git commit -m "Synchronizing completion at $TODAY"
        git push -u origin develop
     fi
}

add_yum_repo() {
cat > /etc/yum.repos.d/google-cloud-sdk.repo <<EOF
[google-cloud-sdk]
name=Google Cloud SDK
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
}

add_apt_source(){
export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
}

install_sdk() {
    local OS_VERSION=$(grep -Po '(?<=^ID=")\w+' /etc/os-release)
    local OS_VERSION=${OS_VERSION:-ubuntu}
    if [[ $OS_VERSION =~ "centos" ]];then
        if ! [ -f /etc/yum.repos.d/google-cloud-sdk.repo ];then
            add_yum_repo
            yum -y install google-cloud-sdk
        else
            echo "gcloud is installed"
        fi
    elif [[ $OS_VERSION =~ "ubuntu" ]];then
        if ! [ -f /etc/apt/sources.list.d/google-cloud-sdk.list ];then
            add_apt_source
            sudo apt-get -y update && sudo apt-get -y install google-cloud-sdk
        else
             echo "gcloud is installed"
        fi
    fi
}

auth_sdk(){
    local AUTH_COUNT=$(gcloud auth list --format="get(account)"|wc -l)
    if [ $AUTH_COUNT -eq 0 ];then
        gcloud auth activate-service-account --key-file=$HOME/gcloud.config.json
    else
        echo "gcloud service account is exsits"
    fi
}

repository_list() {
    if ! [ -f repo_list.txt ];then
        gcloud container images list --repository=${GCR_NAMESPACE} --format="value(NAME)" > repo_list.txt && \
        echo "get repository list done"
    else
        /bin/mv  -f repo_list.txt old_repo_list.txt
        gcloud container images list --repository=${GCR_NAMESPACE} --format="value(NAME)" > repo_list.txt && \
        echo "get repository list done"
        DEL_REPO=($(diff  -B -c  old_repo_list.txt repo_list.txt |grep -Po '(?<=^\- ).+|xargs')) && \
        rm -f old_repo_list.txt
        if [ ${#DEL_REPO} -ne 0 ];then
            for i in ${DEL_REPO[@]};do
                rm -rf ${i##*/}
            done
        fi
    fi
}

generate_changelog(){
    if  ! [ -f CHANGELOG.md ];then
        echo  >> CHANGELOG.md
    fi

}

push_image(){
    GCR_IMAGE=$1
    DOCKERHUB_IMAGE=$2
    docker pull ${GCR_IMAGE}
    docker tag ${GCR_IMAGE} ${DOCKERHUB_IMAGE}
    docker push ${DOCKERHUB_IMAGE}
    echo "$IMAGE_TAG_SHA" > ${IMAGE_NAME}/${i}
    sed -i  "1i\- ${DOCKERHUB_IMAGE}"  CHANGELOG.md
}

clean_images(){
     IMAGES_COUNT=$(docker image ls|wc -l)
     if [ $IMAGES_COUNT -gt 1 ];then
         docker image prune -a -f
     fi
}

clean_disk(){
    DODCKER_ROOT_DIR=$(docker info --format '{{json .}}'|jq  -r '.DockerRootDir')
    USAGE=$(df $DODCKER_ROOT_DIR|awk -F '[ %]+' 'NR>1{print $5}')
    if [ $USAGE -eq 80 ];then
        wait
        clean_images
    fi
}

main() {
    git_init
    install_sdk
    auth_sdk
    repository_list
    generate_changelog
    TODAY=$(today)
    PROGRESS_COUNT=0
    LINE_NUM=0
    LAST_REPOSITORY=$(tail -n 1 repo_list.txt)
    while read GCR_IMAGE_NAME;do
        let LINE_NUM++
        IMAGE_INFO_JSON=$(gcloud container images list-tags $GCR_IMAGE_NAME  --filter="tags:*" --format=json)
        TAG_INFO_JSON=$(echo "$IMAGE_INFO_JSON"|jq '.[]|{ tag: .tags[] ,digest: .digest }')
        TAG_LIST=($(echo "$TAG_INFO_JSON"|jq -r .tag))
        IMAGE_NAME=${GCR_IMAGE_NAME##*/}
        if [ -f  breakpoint.txt ];then
           SAVE_DAY=$(head -n 1 breakpoint.txt)
           if [[ $SAVE_DAY != $TODAY ]];then
             :> breakpoint.txt
           else
               BREAK_LINE=$(tail -n 1 breakpoint.txt)
               if [ $LINE_NUM -lt $BREAK_LINE ];then
                   continue
               fi
           fi
        fi
        for i in ${TAG_LIST[@]};do
            GCR_IMAGE=${GCR_IMAGE_NAME}:${i}
            DOCKERHUB_IMAGE=${DOCKERHUB_NAMESPACE}/${IMAGE_NAME}:${i}
            IMAGE_TAG_SHA=$(echo "${TAG_INFO_JSON}"|jq -r "select(.tag == \"$i\")|.digest")
            if [[ $GCR_IMAGE_NAME == $LAST_REPOSITORY ]];then
                LAST_TAG=${TAG_LIST[-1]}
                LAST_IMAGE=${LAST_REPOSITORY}:${LAST_TAG}
                if [[ $GCR_IMAGE  == $LAST_IMAGE ]];then
                    wait
                    clean_images
                fi
            fi
            if [ -f $IMAGE_NAME/$i ];then
                echo "$IMAGE_TAG_SHA"  > /tmp/diff.txt
                if ! diff /tmp/diff.txt $IMAGE_NAME/$i &> /dev/null ;then
                     clean_disk
                     push_image $GCR_IMAGE $DOCKERHUB_IMAGE &
                     let PROGRESS_COUNT++
                fi
            else
                mkdir -p $IMAGE_NAME
                clean_disk
                push_image $GCR_IMAGE $DOCKERHUB_IMAGE &
                let PROGRESS_COUNT++
            fi
            COUNT_WAIT=$[$PROGRESS_COUNT%50]
            if [ $COUNT_WAIT -eq 0 ];then
               wait
               clean_images
               git_commit
            fi
        done
        if [ $COUNT_WAIT -eq 0 ];then
            wait
            clean_images
            git_commit
        fi

        echo "sync image $MY_REPO/$IMAGE_NAME done."
        echo -e "$TODAY\n$LINE_NUM" > breakpoint.txt
    done < repo_list.txt 
    sed -i "1i-------------------------------at $(date +'%F %T') sync image repositorys-------------------------------"  CHANGELOG.md
    git_commit
}

main

```
###设置travis
![image.png](https://upload-images.jianshu.io/upload_images/3481257-557b2d070bed34f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

设置cronjob
![image.png](https://upload-images.jianshu.io/upload_images/3481257-234c785aa729c127.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

手动触发构建
![image.png](https://upload-images.jianshu.io/upload_images/3481257-5abf8afbabc8806e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

查看job Log
![image.png](https://upload-images.jianshu.io/upload_images/3481257-8772a385ea36190d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终验证是否同步到DockeHub https://hub.docker.com/u/mirrorgcrio/?page=1
![image.png](https://upload-images.jianshu.io/upload_images/3481257-05ff78bfe7bec9b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###用法：
```
docker pull gcr.io/google-containers/federation-controller-manager-arm64:v1.3.1-beta.1
# eq
docker pull mirrorgcrio/federation-controller-manager-arm64:v1.3.1-beta.1

```




