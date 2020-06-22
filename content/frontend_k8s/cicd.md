---
title: "前端使用K8S搭建测试环境 - 创建CI/CD"
date: 2020-06-22T12:48:24+08:00
draft: true
---

CI/CD每一个任务都是一个独立的容器，互不干扰，我们可以创建以下几个任务阶段：

* 项目构建阶段，安装依赖以及构建项目
* 容器构建阶段，对构建好的项目打包成Docker镜像
* 部署K8S环境
* 部署最终环境

上面说过，执行CI/CD时是通过一个独立的容器去执行的，我们没有办法能够直接CI/CD的文件复制到宿主机上，所以我们需要打包阶段来构建镜像，其次我们在CI/CD的容器中，也没有办法直接对宿主机的K8S集群创建pod和service，在这里我们还需要额外制作一个能够在宿主机上创建pod和service的镜像。

### 创建“宿主机执行者”容器

先来普及以下，kubectl能够操作K8S集群，是根据配置和证书而来的，我们通过MINIKUBE启动一个集群的时候，自动就已经帮忙我们做好了这些事，默认我们通过kubectl就是操作当前服务器下的K8S集群，如果我们拿到远程K8S集群的证书和信息，那么我们也可以在本地对远端的K8S集群做操作。

我们创建的这个容器，就是包含了该K8S集群的信息，这样CI/CD的时候就可以使用这个容器对宿主机创建pod和service。

让我们来编写Dockerfile:

```Dockerfile
FROM alpine:latest

RUN mkdir -p /root/.minikube

COPY kubectl /usr/bin/
COPY kube/config /root/.kube/
COPY minikube/* /root/.minikube/
```

我们把kubectl命令复制到容器中，方便使用，同时把~/.kube/config和~/.minikube/下的`ca.crt`、`client.crt`、`client.key`文件复制到容器中。

最后使用命令`docker build -t ${imagename}`来打包镜像。

对于构建好的Docker镜像如果不希望直接发布到外网，可自建一个Docker仓库，这里推荐使用[harbor](https://goharbor.io/)，`harbor`的安装并没有什么坑点，依照官方文档一般都可以顺利安装，这里也不会对`harbor`的使用做详细说明。

镜像制作完成后我们需要推送到私有仓库，推送以前我们需要先给镜像打上tag。

要推送到`harbor`的话，tag需要遵循server/group/imagename的规则，既比如我们的`harbor`访问地址是`harbor.mydocker.com`，项目分组是`frontend`，那么这个标签就应该写成：`harbor.mydocker.com/frontend/imagename`

接着我们登陆docker:
```bash
$ docker login harbor.mydocker.com

> 输入用户名和密码
> 显示成功

$ docker push harbor.mydocker.com/frontend/imagename
```

最终等镜像推送成功后，我们应该可以在`harbor`中查看到镜

### 配置CI/CD脚本

这里不对gitlab-ci的配置展开详细说明，[官方文档](https://docs.gitlab.com/ee/ci/yaml/)的介绍已经非常详细。

下面我们尝试对开头我们描述的5个阶段进行Demo配置：

```yml
# 这里就是我们的 宿主机执行者 镜像
image: harbor.com/frontend/gaia_builder:0.0.1

# 对应开头所说的4个阶段
stages:
  - build_project
  - build_docker
  - deploy_k8s
  - deploy_server

build_project:
  # 我们构建项目时所用到的环境以及工具
  # 比如nodejs版本，yarn等工具，都由这个镜像提供
  # 这个镜像后面会说到具体的制作方法
  image: harbor.com/frontend/frontend_toolkit:0.0.1
  # 执行时机改为手动，我们并不希望每次commit的时候都会自动构建出一个环境
  when: manual
  stage: build_project
  # 将打包资源作为附件往下个阶段传递
  artifacts:
    paths:
      - dist
  # 希望使用Runner的Tag
  tags:
    - fek8s
  script:
    # 执行具体的构建脚本，根据实际情况做调整即可
    # 最终我们会删除node_modules以减少我们服务镜像的体积
    - yarn install --forzen-lockfile
    - yarn build
    - rm -rf node_modules/

build_docker:
  image: docker:latest
  when: manual
  # 开头我们配置的Runner就是在Docker下执行的，所以这里当我们要打包Docker镜像的时候，相当于是在Docker下执行Docker
  # 下面的配置可以理解为标配
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ''
    DOCKER_HOST: tcp://127.0.0.1:2375
    # 导出一些通用的环境变量，方便后面重复使用
    DOCKER_IMAGE_NAME: $GITLAB_USER_LOGIN-$CI_BUILD_REF_NAME
    DOCKER_IMAGE_TAG: harbor.com/myproject/$GITLAB_USER_LOGIN-$CI_BUILD_REF_NAME:latest
  services:
    # dind是docker in docker的缩写
    # 通过--insecure-registry参数指定，我们私有仓库的代码是合法的
    - name: docker:dind
      command: ["--insecure-registry=harbor.com"]
  stage: build_docker
  dependencies:
    - build_project
  tags:
    - fek8s
  script:
    # 这个镜像后面会说到，它是服务的基础镜像
    - export CACHE_IMAGE=harbor.com/frontend/gaia_nginx:latest
    # 拉取镜像，并抑制因stderr的输出导致ci/cd执行失败
    - docker pull $CACHE_IMAGE || true
    # 构建服务镜像
    - docker build --cache-from $CACHE_IMAGE -t $DOCKER_IMAGE_TAG -f gaia/Dockerfile .
    # 登录私有化Docker服务
    # 考虑到不同的用户，登录账号密码都不同，所以将其做成变量
    # 可通过Gitlab项目设置中CI/CD中添加变量来保护
    - docker login -u $DOCKER_USERNAME -p $DOCKER_PWD harbor.com
    # 最终推送服务镜像到服务器
    - docker push $DOCKER_IMAGE_TAG

deploy_k8s:
  when: manual
  stage: deploy_k8s
  tags:
    - fek8s
  script:
    # 由于service的名字无法使用下划线，而我们的分支名经常会使用到下划线，所以将其替换为横线防止错误
    - export K8S_SVC_BRANCH=$(echo $CI_BUILD_REF_NAME | sed 's/_/-/g')
    # 为了减少环境构建成本，服务名固定为`${username}-${branch}-svc`的格式
    - export SVC_NAME=$GITLAB_USER_LOGIN-$K8S_SVC_BRANCH-svc
    # 删除老的服务
    - kubectl delete svc $SVC_NAME --ignore-not-found --namespace=default
    # 删除老的部署
    - kubectl delete deploy $SVC_NAME --ignore-not-found --namespace=default
    # 创建新的服务
    - kubectl run $SVC_NAME --image=harbor.com/myproject/$GITLAB_USER_LOGIN-$CI_BUILD_REF_NAME:latest --namespace=default
    # 部署服务，并暴露80端口，方便K8S服务器队其转发
    - kubectl expose deploy $SVC_NAME --port=80 --namespace=default

deploy_server:
  image: curlimages/curl:latest
  stage: deploy_server
  tags:
    - fek8s
  needs: ["deploy_k8s"]
  script:
    # 调用宿主机（也就是K8S服务器）的服务，来重载nginx转发
    # 这类的服务后面会进行说明
    - curl -X POST 10.13.82.219:3500/svc
```

至此我们的ci脚本已配置完毕，接下来我们完成本文遗留下来的一些基础镜像和服务。