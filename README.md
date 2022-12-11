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
cat <<EOF | kubectl apply -n default -f -
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

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: gogit
spec:
  entrypoint: main                  # 执行入口
  arguments:
    # 工作流全局参数
    parameters:
      - name: repo
        value: https://github.com/linuxsuren/gogit
      - name: branch
        value: master

  # Volume 模板申明，用于工作流中多个 Pod 之间共享存储
  # 例如：克隆代码、构建代码的 Pod 之间共享目录
  # 动态创建 Volume，与当前工作流的生命流程保持一致
  volumeClaimTemplates:
    - metadata:
        name: work
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 64Mi

  templates:
  - name: main
    dag:
      tasks:
        - name: clone
          template: clone   # 引用下面的模板，并传入参数
          arguments:
            parameters:
              - name: repo
                value: "{{workflow.parameters.repo}}"
              - name: branch
                value: "{{workflow.parameters.branch}}"
        - name: build
          template: build
          depends: checkout # 通过 depends 设置执行任务之间的顺序关系
  
  - name: clone
    inputs:
      parameters:
        - name: repo
        - name: branch
    container:
      volumeMounts:
        - mountPath: /work              # 共享该目录
          name: work
      image: alpine/git:v2.26.2
      workingDir: /work                 # 代码会克隆到这个目录中
      args:
        - clone
        - --depth
        - "1"
        - --branch
        - "{{inputs.parameters.branch}}"
        - --single-branch
        - "{{inputs.parameters.repo}}"
        - .
  - name: build
    container:
      image: golang:1.19
      volumeMounts:
        - name: work
          mountPath: /work
      workingDir: /work
      env:
        - name: GOPROXY     # 根据需要来设置相应的环境变量，例如这里的 Golang 代理
          value: https://goproxy.io,direct
      command:
        - make
      args:
        - build             # 执行 Makefile 中的 build 指令来构建 Golang 代码
EOF
```

### 小结

通过上面的例子，我们可以看到 Argo Workflows 的如下特点：

* 一个工作流中的多个任务之间默认是并行执行的，如果希望顺序执行则可以通过 `depends` 设置
* 工作流中可以申明多个任务模板，只有被 `entrypoint` 引用到的模板才会被执行
* 每执行一个任务都会对应启动一个 Pod
* 一个工作流之间的多个任务需要共享目录的话，需要挂载 Volume

## Webhook
所有主流 Git 仓库都是支持 webhook 的，借助 webhook 可以当代码发生变化后实时地触发工作流的执行。

Argo Workflows 利用 `WorkflowEventBinding` 将收到的 webhook 请求与 WorkflowTemplate 做关联。请参考下面的例子：

```shell
cat <<EOF kubectl apply -n default -f -
apiVersion: argoproj.io/v1alpha1
kind: WorkflowEventBinding
metadata:
  name: pull-request-binding
spec:
  event:
    # 通过 webhook 的 payload 对请求进行过滤，并联动触发对应的工作流模板
    selector: payload.project.name == "gogit" && payload.object_attributes.state == "opened"
  submit:
    workflowTemplateRef:
      name: gogit       # 关联工作流模板
    arguments:
      parameters:
      - name: branch
        valueFrom:
          # 从 webhook 的 payload 中提取值作为参数
          event: payload.object_attributes.source_branch
EOF
```

然后，在代码仓库中添加如下的 webhook 地址（其中，`default` 是 `WorkflowEventBinding` 所在的命名空间）：

```
https://argo-workflow-ip:port/api/v1/events/default/
```

### 小结

从上面的例子中，我们可以看到：

* Argo Workflows 以申明式的资源将 webhook 与工作流模板做关联，非常地灵活
* webhook 绑定并不局限在 Git 代码仓库上，还可以与其他类型的 webhook 做关联
* 通过从 webhook 的 payload 中提取值，可以非常方便地为工作流模板参数传递值

## 关联代码仓库

对于不少的团队而言，会出于各种考虑而选择私有部署 Git 服务，例如：Gitlab、Gitee 等。而将工作流的执行结果与代码仓库的 Pull Request 相关联几乎是一个**标配**。以下是关联后的几点好处：

* 工作流执行失败后阻止 Pull Request 的合并
* 在 Pull Request 页面中可以直接看到工作流执行状态

下面会基于 https://github.com/LinuxSuRen/gogit/ 给出一个关联方案：

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: v1
data:
  token: eW91ci10b2tlbg==
kind: Secret
metadata:
  name: git-secret
type: Opaque
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hook
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: pr
        value: 1
      - name: argoServer
        value: https://localhost:8080

  hooks:
    exit:                   # 只有 exit 这个 hook 名称是固定的
      template: hook
    all:                    # 这里可以是任意字符串，重点在于 expression 这里的表达式
      template: hook
      expression: "true"    # 可以通过表达式 expression 对事件进行过滤

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
  - name: hook
    volumes:
      - name: git-secret
        secret:
          defaultMode: 0400
          secretName: git-secret    # 包含 token 字段的 Secret
    container:
      image: ghcr.io/linuxsuren/gogit:master@sha256:4855f4ffbc1644eb7246f94cc9ee12c793ed4c26ba18e1d4d9afa57b72f1e846
      args:
        - --provider=github         # 支持 GitHub、Gitlab 等，私有部署的话需要参数 --server 指定地址
        - --owner=LinuxSuRen        # 根据需要修改 owner、repo、username
        - --repo=gogit
        - --username=LinuxSuRen
        - --token=file:///root/.ssh/token
        - --pr={{workflow.parameters.pr}}
        - --target={{workflow.parameters.argoServer}}/workflows/{{workflow.namespace}}/{{workflow.name}}
        - --status={{workflow.status}}
      volumeMounts:
        - mountPath: /root/.ssh/
          name: git-secret
EOF
```

触发上面的工作流后，就会在指定的仓库 Pull Request 上出现构建状态。

### 参考链接

* [官方文档](https://argoproj.github.io/argo-workflows/lifecyclehook/)

### 小结

从这个示例中，我们可以看到：

* hook 机制依然是非常的灵活，但 expression 表达式可能会是一个具有挑战的部分
* hook 机制有点像是 Golang 的 `__init` 函数，作为特殊的入口，可以调用其他的模板

## 日志持久化
Argo Workflows 默认不会持久化工作流日志，而是从每个任务对应的 Pod 中获取日志。而对于 Kubernetes 来说，Pod 是一个没有保障的最小执行单元，可能会由于人为或者某种策略被删除。当 Pod 被删除后，日志就无法查看了。因此，对于生产环境而言，必须要持久化日志。

Argo Workflows 执行多种存储协议，以下是兼容 S3 的 MinIO 存储：

首先，下载、安装以及配置 minio。本文仅作学习、演示使用，生产环境中，请按照官方文档进行安装、配置。

```shell
hd i minio
minio server /tmp/minio --console-address ":9001"
```

然后，访问 minio 管理界面 `http://localhost:9001`，创建名为 `argo-workflow` 的 `bucket`。创建 `Access Key`，并写入下面的 `Secret` 中。

安装如下配置修改 `ConfigMap`：

```shell
kubectl create secret generic minio-workflow \
  --from-literal=accessKey=supersecret \
  --from-literal=secretKey=topsecret
cat <<EOF | kubectl apply -n default -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
data:
  artifactRepository: |
    archiveLogs: true          # 全局设置使得所有工作流日志做持久化
    s3:
      bucket: argo-workflow    # 在 minio 中创建的 bucket
      endpoint: minio.minio.svc
      insecure: true                  # 当 minio 没有启用 TLS
      accessKeySecret:
        name: minio-workflow
        key: accessKey
      secretKeySecret:
        name: minio-workflow
        key: secretKey
EOF
```

完成上面的配置后，再次执行任意工作流，并将执行完成的 Pod 删除后，我们依然可以在 UI 上查看任务日志。并且，可以在 minio 中看到了新增的文件。每个 Pod 的日志在 minio 中分别以一个文件的形式存储。

我们可以通过 minio 的命令行客户端 `mc` 看到类似如下的文件：

```shell
mc alias set myminio http://localhost:9001 minioadmin minioadmin
# mc ls myminio/argo-workflow -r
[2022-12-09 10:53:31 CST]    20B STANDARD hello-world-5mjgp/hello-world-5mjgp-clone-3848310779/main.log
[2022-12-09 10:55:39 CST]  16KiB STANDARD hello-world-5mjgp/hello-world-5mjgp-image-2614052838/main.log
[2022-12-09 10:54:35 CST] 5.9KiB STANDARD hello-world-5mjgp/hello-world-5mjgp-scan-4101005739/main.log
[2022-12-09 10:53:51 CST]   435B STANDARD hello-world-5mjgp/hello-world-5mjgp-test-1532501286/main.log
```

### 参考链接

* [支持的外部存储类型](https://argoproj.github.io/argo-workflows/configure-artifact-repository/)
* [官方文档](https://argoproj.github.io/argo-workflows/configure-archive-logs/)

### 小结

通过上面的例子，我们可以看到：

* Argo Workflows 能以非侵入式的配置，使得工作流日志输出到对象存储等外部存储中
* Argo Workflows 的任务有输入、输出（input、output）的概念，日志的持久化是将日志作为输出写入到预先配置好的外部存储
* 日志的持久化，可以分别在全局 ConfigMap、Workflow Spec、WorkflowTemplate 中配置

## 引用已有模板中的任务

Argo Workflows 允许以三种方式引用已有的任务：

* 当前工作流
* 当前命名空间中的工作流模板
* 全局（Cluster 级别）的工作流模板

这相当于 Java、Golang 等编程语言中的引用方式，分别可以引用：当前源文件、当前包、其他包下的函数。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world
spec:
  templates:
  - name: hd
    dag:
      tasks:
      - name: hd
        templateRef:            # 表示引用其他模板中的任务
          name: hook            # 模板名称
          template: hook        # 模板中的任务名称
          clusterScope: true    # 为 true 是从全局（Cluster）中查找模板，为 false 时从当前命名空间中查找
        arguments:
          parameters:           # 给所引用的任务传递参数
          - name: pr
            value: 1
```

### 小结

我们可以将公用的模板作为模板库，供工作流调用，这样就可以使得工作流变得简单。

## SSO
TODO

## 插件机制
TODO

## References
* [DevOps Practice Guide](https://github.com/LinuxSuRen/devops-practice-guide)
