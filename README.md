[Argo Workflows](https://argoproj.github.io/argo-workflows/) 是一个[云原生](https://en.wikipedia.org/wiki/Cloud-native_computing)的通用的工作流引擎。本教程主要介绍如何用其完成持续集成（Continous Integration, CI）任务。

## 基本概念
对任何工具的基本概念有一致的认识和理解，是我们学习以及与他人交流的基础。

以下是本文涉及到的概念：

* WorkflowTemplate，工作流模板
* Workflow，工作流

为方便读者理解，下面就几个同类工具做对比：

| Argo Workflow | Jenkins |
|---|---|
| WorkflowTemplate | Pipeline |
| Workflow | Build |

## 安装
首先，你需要有一套 [Kubernetes](https://github.com/kubernetes/kubernetes/) 环境。下面的工具可以帮助你快速按照好一套 Kubernetes 环境：

> 推荐使用 [hd](https://github.com/LinuxSuRen/http-downloader) 安装下面的工具
>
> 安装 `hd` 的命令为：`curl https://linuxsuren.github.io/tools/install.sh|bash`

| 工具 | 工具安装 |使用 |
|---|---|---|
| [k3d](https://k3d.io/) | `hd i k3d` | `k3d cluster create` |
| [kubekey](https://github.com/kubesphere/kubekey) | `hd i kk` | `kk create cluster` |
| [minikube](https://github.com/kubernetes/minikube) | `hd i minikube` | `minikube start` |

当 Kubernetes 环境就绪后，就可以通过下面的命令会在命名空间（`argo`）下安装最新版本的 `Argo Workflow`：

```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

如果你的环境访问 GitHub 时有网络问题，可以使用下面的命令来安装：

```shell
docker run -it --rm -v /root/.kube/:/root/.kube --network host ghcr.io/linuxsuren/argo-workflows-guide:master
```

推荐使用的工具：

|||
|---|---|
| [k9s](https://k9scli.io/) | K9s is a terminal based UI to interact with your Kubernetes clusters. |

## 设置访问方式
我们可以用下面的方式或者其他方式来设置 Argo Workflows 的访问端口：

```shell
kubectl -n argo port-forward deploy/argo-server --address 0.0.0.0 2746:2746
```

> 需要注意的是，这里默认的配置下，服务器设置了自签名的证书提供 HTTPS 服务，因此，确保你使用 `https://` 协议进行访问。

例如，地址为：`https://10.121.218.242:2746/`

Argo Workflows UI 提供了多种认证登录方式，对于学习、体验等场景，我们可以通过下面的命令直接设置绕过登录：

```shell
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
```

## 简单示例

下面是一个非常简单的示例：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world
spec:
  entrypoint: hd # 执行入口，类似于 Go、Java 语言的的 main 函数
  templates:
  - container:
      args:
      - search
      - kubectl
      command:
      - hd
      image: ghcr.io/linuxsuren/hd:v0.0.70 # 任务镜像
      name: main
    name: hd
EOF
```

执行成功后，就可以在下面的地址访问到刚刚创建的工作流模板：

`https://10.121.218.242:2746/workflow-templates/default`

选择其中一个模板，点击 `SUBMIT` 按钮，并设置对应的参数后即可触发工作流的执行。

在 Workflows 的详情页面中，我们做如下的操作：

* RESUBMIT，使用相同的模板以及参数触发一次新的执行

### 小结
通过前面的步骤，我们可以观察到 Argo Workflow 有如下特点：

* 需要具备基本的容器知识
* 需要熟悉 Kubernetes 的基本资源，例如：PodTemplate、ConfigMap、Secret、Volume 等
* 实现特定任务，基本上是需要寻找官方提供的镜像，或自行构建镜像

## 完整示例
TODO

## Webhook
TODO

## 集成代码仓库
TODO

## 日志持久化
TODO

## SSO
TODO

## References
* [DevOps Practice Guide](https://github.com/LinuxSuRen/devops-practice-guide)
