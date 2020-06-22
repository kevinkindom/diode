---
title: "前端使用K8S搭建测试环境 - K8S环境搭建"
date: 2020-06-22T12:07:00+08:00
draft: true
---

> 本文不会对K8S细节做说明，只会针对用到K8S的功能做简单的介绍。

由于只是搭建开发环境，所以不会用到多个Pods，在环境的搭建上本着怎么方便怎么来的原则进行。

K8S提供了[minikube](https://kubernetes.io/zh/docs/tasks/tools/install-minikube/) 这样的一个快速的迷你环境搭建，麻雀虽小，但是已经能够满足此次我们环境的搭建了。

`minikube`的安装非常简单，通过链接[下载](https://github.com/kubernetes/minikube/releases)适合自己系统的二进制，并解压到/usr/bin目录即可。

然后执行下面命令启动单节点的k8s集群:
```bash
# vm-driver参与用于指定k8s组件的运行环境，比如在windows下我们可以让k8s的容器跑在vmware、virtualbox等
# 目前我们只需要在当前系统下运行，指定none即可
minikube start --vm-driver=none
```

上述命令执行后，就会自动拉取相关的镜像，如果如果顺利最后可以看到`success`字眼。

由于有墙的存在，如果不能够顺利启动或者拉取所需镜像的话，我们可以手动从国内的docker源进行镜像拉取，之后通过`docker tag`命令给它重新打上和`minikube`所需要镜像同样的tag，完成后重新执行启动命令即可。

`minikube`安装完成后，我们还需要手动安装[kubectl](https://github.com/kubernetes/kubectl/releases)、[kubelet](https://github.com/kubernetes/kubelet)、[kubeadm](https://github.com/kubernetes/kubeadm)，值得注意的是我们需要对应版本下载，截至到这篇文章为止，我们使用`v0.16.7`这个版本比较方便。上面几个工具下载后也可以放置到`/usr/bin`目录下即可。

至此我们K8S部分所需要的基础环境已经搭建完成。

下面我们会将K8S集成到Gitlab上。