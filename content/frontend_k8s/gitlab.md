---
title: "前端使用K8S搭建测试环境 - Gitlab集成K8S"
date: 2020-06-22T12:42:17+08:00
draft: true
---

有点难以置信，上面我们似乎轻易的就部署了k8s集群。接下来在Gitlab中要集成K8S的话，有几项信息必须要填写：

![](https://i.loli.net/2020/06/22/IAsHFQi2Utpn7jK.png)

**集群名称：** 我们可以随意填写

**API地址：** 通过执行命令`kubectl cluster-info`后，从`master`的地址中取得

**CA证书和服务令牌**
要获得这两个信息，我们得要先在K8S中创建一个新得`namespace`，执行命令：`kubectl create ns gitlab`即可

然后我们在部署阶段要去创建，删除资源等，所以我们需要对象得**RBAC**权限，复制下面配置保存如ServiceAccount.yaml：
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab
  namespace: gitlab

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab
  namespace: gitlab
subjects:
  - kind: ServiceAccount
    name: gitlab
    namespace: gitlab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

执行命令：
```bash
kubectl apply -f ServiceAccount.yaml

> serviceaccount "gitlab" created
> clusterrolebinding.rbac.authorization.k8s.io "gitlab" created
```

创建成功后，我们可以使用刚创建得ServiceAccount获取CA证书和服务令牌。

```bash
kubectl get serviceaccount gitlab -n gitlab -o json
# 我们找到`secrets[0].name`这个节点得信息，得到服务名，假设叫gitlab-token-abcde

kubectl get secret gitlab-token-abcde -n gitlab -o json

# 从data[ca.crt]中得到CA证书
# 从data.token得到服务令牌
# 上面两个信息是被base64编码了，解压一下即可得到我们所需要得所有信息
```

得到所有信息后，我们按照表单以此填入即可将K8S集成到Gitlab。

目前我们还需要注册一些Runner来执行CI/CD的任务。

我们可以直接在一个服务器上按照Gitlab官方得说明去添加Runner，但既然我们已经部署了K8S，那么可以在K8S上自动部署上Runner。

在Gitlab中以此访问`运维/Kubernetes`，打开刚添加的K8S，在应用程序中，我们先自动安装`Helm Tiller`，等待`Helm Tiller`安装完成后，我们再安装`Gitlab Runner`，当全部完成后，我们应该可以在`CI/CD`中的`Runner`发现已激活的`Runner`了。

此时Gitlab Runner也配置完成。