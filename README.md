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

## 简单示例
TODO

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
