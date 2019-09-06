# What happens when I type kubectl run?

> 为了确保整体的简单性和易上手，Kubernetes 通过一些简单的抽象隐去操作背后的复杂逻辑，但作为一名有梦想的工程师，掌握其背后的真正思路是十分有必要的。本文以 Kubectl 创建 Pod 为例，向你揭露从客户端到 Kubelet 的请求的完整生命周期。

想象一下，当你想在 Kubernetes 集群部署 Nginx 时，你会执行以下命令：

```bash
kubectl run nginx --image=nginx --replicas=3
```

几秒后，你将看到三个 Nginx Pod 分布在集群工作节点上。这相当神奇，但它背后究竟发生了什么？

Kubernetes 是一个神奇的框架，它通过用户友好（user-friendly）的 API 处理跨基础架构的 Workload 部署。通过简单的抽象隐藏了背后的复杂性。但是，为了充分理解它为我们提供的价值，我们需要理解它的原理。

本指南将带领你充分了解从 Kubectl 客户端到 Kubelet 请求的完整生命周期，并在必要时通过源代码解释它到底是什么。

**注**：本文所有内容基于 `Kubernetes v1.14.0`。

目录
=================

   * [What happens when I type kubectl run?](#what-happens-when-i-type-kubectl-run)
      * [Kubectl](#kubectl)
         * [Validation and generators](#validation-and-generators)
         * [API groups and version negotiation](#api-groups-and-version-negotiation)
         * [Client auth](#client-auth)
      * [kube-apiserver](#kube-apiserver)
         * [Authentication](#authentication)
         * [Authorization](#authorization)
         * [Admission Controller](#admission-controller)
      * [etcd](#etcd)
      * [Control loops](#control-loops)
         * [Deployment Controller](#deployment-controller)
         * [ReplicaSet Controller](#replicaset-controller)
         * [Informers](#informers)
         * [Scheduler](#scheduler)
      * [Kubelet](#kubelet)
         * [Pod Sync](#pod-sync)
         * [CRI and pause container](#cri-and-pause-container)
         * [CNI and pod networking](#cni-and-pod-networking)
         * [Inter-host networking](#inter-host-networking)
         * [Container startup](#container-startup)
      * [Wrap-up](#wrap-up)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

## Kubectl

### Validation and generators

首先，当我们敲下回车键执行命令后， Kubectl 会执行客户端验证，以确保非法的请求（例如，创建不支持的资源或使用[格式错误的镜像名称](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L264)）快速失败，并不会发送给 kube-apiserver，即通过减少不必要的负载来提高系统性能。

验证通过后， Kubectl 开始构造它将发送给 kube-apiserver 的 HTTP 请求。在 Kubernetes 中，访问或更改状态的所有尝试都通过 kube-apiserver 进行，​​后者又与 etcd 进行通信。 Kubectl 客户端也不例外。为了构造 HTTP 请求， Kubectl 使用称为 [generators](https://kubernetes.io/docs/user-guide/kubectl-conventions/#generators) 的东西，这是一个负责序列化的抽象概念。

你可能没有注意到，通过 `kubectl run` 不仅可以运行 `deployment`，还可以通过指定参数 `--generator` 来部署其它 workload。

如果没有指定 `--generator` 参数的值， Kubectl 将会自动[推断](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L319-L339)资源的类型，具体如下：

- 具有 `--restart-policy=Always` 的资源被视为 Deployment；
- 具有 `--restart-policy=OnFailure` 的资源被视为 Job；
- 具有 `--restart-policy=Never` 的资源被视为 Pod。

Kubectl 还将确定是否需要触发其他操作，例如记录命令（用于部署或审计），或者此命令是否是 dry run。

> From wikipedia
>
> 空运行（dry run）也称为试运行（practice run），是刻意为了减轻可能失效的影响而有的测试流程。例如飞机公司会先在飞机停在陆地上时进行其弹射座椅的测试，之后才在飞机升空后主进行类似测试。陆地上的测试即为空运行。
>
> 在验收程序（也称为工厂验收测试）的领域中，空运行是指分包商需在产品交给客户，进行真正的验收测试之前，先进行的完整测试。

当 Kubectl 判断出要创建一个 Deployment 后，它将使用 `DeploymentV1Beta1 generator` 配合我们提供的参数，生成一个[运行时对象（Runtime Object）](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/generate/versioned/run.go#L237)。

### API groups and version negotiation

这里值得指出的是， Kubernetes 使用的是一个分类为 API Group 的版本化 API。它旨在对资源进行分类，以便于推理。

同时，它还为单个 API 提供了更好的版本化方案。 Deployment 的 API Group 为 `apps`，其最新版本为 `v1`。这就是为什么需要在 Deployment manifests 顶部指定 `apiVersion: apps/v1` 的原因。

回归正文， Kubectl 生成运行时对象之后，它开始为它[查找合适的 API Group 和版本](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L674-L686)，然后组装一个知道该资源的各种 REST 语义的[版本化客户端](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L705-L708)。

这个发现阶段称为版本协商 (version negotiation)，涉及 Kubectl 扫描 remote API 上的 `/apis` 路径以检索所有可能的 API Group。

由于 kube-apiserver 在 `/apis` 路径中暴露其 OpenAPI 格式的 scheme 文档，因此客户端可以轻松的找到匹配的 API。

为了提高性能， Kubectl 还将 [OpenAPI scheme 缓存到 `~/.kube/cache/discovery` 目录](https://github.com/kubernetes/kubernetes/blob/v1.14.0/staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go#L234)。如果要了解 API 发现的完整过程，你可以尝试删除该目录并在运行 Kubectl 命令时将 `-v` 参数的值设为最大，然后你将会在日志中看到所有试图找到这些 API 版本的 HTTP 请求。

最后一步才是真正地[发送 HTTP 请求](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L709)。一旦请求获得成功的响应， Kubectl 将会根据所需的[输出格式](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L459)打印 success message。

### Client auth

我们在上文中没有提到的一件事是客户端身份验证（这是在发送 HTTP 请求之前处理的），现在让我们来看看。

为了成功发送请求， Kubectl 需要先进行身份验证。用户凭据一般存储在 `kubeconfig` 文件中，但该文件可以存储在不同的位置。为了定位到它， Kubectl 执行以下操作：

- 如果指定参数 `--kubeconfig`，那么采用该值；
- 如果指定环境变量 `$KUBECONFIG`，那么采用该值；
- 否则[查看默认的目录](https://github.com/kubernetes/client-go/blob/kubernetes-1.14.0/tools/clientcmd/loader.go#L52)，如 `~/.kube`，并使用找到的第一个文件。

解析文件后，它会确定当前要使用的上下文，当前指向的集群以及与当前用户关联的所有身份验证信息。如果用户提供了额外的参数（例如 `--username`），则这些值优先，并将覆盖 kubeconfig 中指定的值。

一旦有了上述信息， Kubectl 就会填充客户端的配置，以便它能够适当地修饰 HTTP 请求：

- x509 证书使用 [`tls.TLSConfig`](https://github.com/kubernetes/client-go/blob/kubernetes-1.14.0/rest/transport.go#L80-L89) 发送（包括 CA 证书）；
- bearer tokens 在 HTTP 请求头 Authorization 中[发送](https://github.com/kubernetes/client-go/blob/kubernetes-1.14.0/transport/round_trippers.go#L316)；
- 用户名和密码通过 HTTP 基础认证[发送](https://github.com/kubernetes/client-go/blob/kubernetes-1.14.0/transport/round_trippers.go#L197)；
- OpenID 认证过程是由用户事先手动处理的，产生一个像 bearer token 一样被发送的 token。

## kube-apiserver

### Authentication

我们的请求已经发送成功，接下来呢？kube-apiserver！

kube-apiserver 是客户端和系统组件用来持久化和检索集群状态的主要接口。为了执行其功能，它需要能够验证请求是否合法。此过程称为认证 （Authentication）。

为了验证请求，当服务器首次启动时， kube-apiserver 会查看用户提供的所有 [CLI 参数](https://v1-14.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)，并组装合适的 authenticator 列表。

举个例子：

- 如果指定参数 `--client-ca-file`，它会附加 x509 authenticator 到列表中；
- 如果指定参数 `--token-auth-file`，它会附加 token authenticator 到列表中。

每次收到请求时，都会[遍历身份验证器列表](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/authentication/request/union/union.go#L53)，直到成功为止：

- [x509 handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/authentication/request/x509/x509.go#L89) 会验证 HTTP 请求是否是通过 CA 根证书签名的 TLS 密钥编码的；
- [bearer token handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/authentication/request/bearertoken/bearertoken.go#L37) 会验证 HTTP Authorization header 指定的 token 是否存在于 `--token-auth-file` 参数提供的 token 文件中；
- [basicauth handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/plugin/pkg/authenticator/request/basicauth/basicauth.go#L39) 会简单验证 HTTP 请求的基本身份凭据。

如果所有 authenticator 都认证失败，则请求失败并返回汇总的错误信息。

如果认证成功，则会从请求中删除 `Authorization` 标头，并[将用户信息添加到其上下文中](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/filters/authentication.go#L74-L77)。为之后的操作（例如授权和准入控制器）提供访问先前建立的用户身份的能力。

### Authorization

好的，请求已发送，kube-apiserver 已成功验证我们是谁。终于解脱了？！

想太多！

虽然我们证明了自己是谁，但还没证明有权执行此操作。毕竟，身份 (identity) 和许可 (permission) 并不是一回事。因此 kube-apiserver 需要授权。

kube-apiserver 处理授权的方式与身份验证非常相似：基于 [CLI 参数](https://v1-14.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) 输入，汇集一系列 authorizer， 这些 authorizer 将针对每个传入请求运行。如果所有 authorizer 都拒绝该请求，则该请求将导致 `Forbidden` 响应并且[不再继续](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/filters/authorization.go#L76)。如果单个 authorizer 批准，则请求继续。

Kubernetes v1.14 的 authorizer 实例：

- [webhook](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/plugin/pkg/authorizer/webhook/webhook.go#L152)：与集群外的 HTTP(S) 服务交互；
- [ABAC](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/auth/authorizer/abac/abac.go#L224)：执行静态文件中定义的策略；
- [RBAC](https://github.com/kubernetes/kubernetes/blob/v1.14.0/plugin/pkg/auth/authorizer/rbac/rbac.go#L74)：执行由集群管理员添加为 k8s 资源的 RBAC 规则；
- [Node](https://github.com/kubernetes/kubernetes/blob/v1.14.0/plugin/pkg/auth/authorizer/node/node_authorizer.go#L80)：确保 kubelet 只能访问自己节点上的资源。

### Admission Controller

好的，到目前为止，我们已经过认证并获得了 kube-apiserver 的授权。那接下来呢？

从 kube-apiserver 的角度来看，它相信我们是谁并允许我们继续，但是对于 Kubernetes， 系统的其他组件对应该和不应该允许发生的内容有异议。所以 [Admission Controller](https://v1-14.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 该闪亮登场了。

虽然 Authorization 的重点是回答用户是否具有权限，但是 Admission Controllers 仍会拦截该请求，以确保其符合集群的更广泛期望和规则。它们是对象持久化到 etcd 之前的最后一个堡垒，因此它们封装了剩余的系统检查以确保操作不会产生意外或负面结果。

Admission Controller 的工作方式类似于 Authentication 和 Authorization 的工作方式，但有一个区别：如果单个 Admission Controller 失败，整个链断开，请求将失败。

Admission Controller 设计的真正优势在于它致力于提升*可扩展性*。每个控制器都作为插件存储在 [plugin/pkg/admission](https://github.com/kubernetes/kubernetes/tree/v1.14.0/plugin/pkg/admission) 目录中，最后编译进 kube-apiserver 二进制文件。

Kubernetes 目前提供十多种 Admission Controller，此处建议阅读文档 [Kubernetes Admission Controller](https://v1-14.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)。

## etcd

到目前为止， Kubernetes 已经完全审查了传入的请求，并允许它往下走。在下一步中，kube-apiserver 将反序列化 HTTP 请求，构造运行时对象（runtime object）（有点像 kubectl generator 的逆过程），并将它们持久化到 etcd。

这里插入一下，kube-apiserver 是怎么知道在接受我们的请求时该怎么做呢？

在提供任何请求之前，kube-apiserver 会发生一系列非常复杂的步骤。让我们从第一次运行 kube-apiserver 二进制文件开始：

1. 当运行 kube-apiserver 二进制文件时，它会创建一个[服务链](https://github.com/kubernetes/kubernetes/blob/v1.14.0/cmd/kube-apiserver/app/server.go#L157)，允许 apiserver 聚合。这是一种支持多 apiserver 的方式；
1. 之后，它会创建一个用作默认实现的 [generic apiserver](https://github.com/kubernetes/kubernetes/blob/v1.14.0/cmd/kube-apiserver/app/server.go#L179)；
1. 使用生成的 OpenAPI scheme 填充 [apiserver 配置](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/server/config.go#L147)；
1. 然后，kube-apiserver 遍历 scheme 中指定的所有 API Group， 并为其构造 [storage provider](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/master/master.go#L420)。当你访问或变更资源状态时， kube-apiserver 就会调用这些 API Group；
1. 对于每个 API Group， 它还会迭代每个组版本，并为每个 HTTP 路由[安装 REST 映射](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/groupversion.go#L99)。这允许 kube-apiserver 映射请求，并且一旦找到匹配就能够委托给正确的代码逻辑；
1. 对于本文的特定用例，将注册一个 [POST handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/installer.go#L737)，该处理程序将委托给 [create resource handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/handlers/create.go#L46)。

到目前为止， kube-apiserver 完全知道存在哪些路由及内部映射，当请求匹配时，可以知道调用哪些处理程序和存储程序。这是非常完美的设计模式。这里我们假设 HTTP 请求已经被 kube-apiserver 收到了：

1. 如果程序处理链可以将请求与注册的路由匹配，它会将该请求交给注册到该路由的 [dedicated handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/server/handler.go#L146)。否则它会回退到 [path-based handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/server/mux/pathrecorder.go#L248)（这是调用 `/apis` 时会发生的情况）。如果没有为该路由注册处理程序，则会调用 [not found handler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/server/mux/pathrecorder.go#L254)，最终返回 `404`；
1. 幸运的是，我们有一个处理器名为 [createHandler](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/handlers/create.go#L46)! 它有什么作用？它将首先解码 HTTP 请求并执行基础验证，例如确保请求提供的 JSON 与我们的版本化 API 资源匹配；
1. [审计和准入控制阶段](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/handlers/create.go#L126-L138)；
1. 然后，资源会通过 [storage provider](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/registry/generic/registry/store.go#L359) [存储到 etcd 中](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/handlers/create.go#L156-L161)。默认情况下，保持到 etcd 的键的格式为 `<namespace>/<name>`，当然，它也支持自定义；
1. 资源创建过程中出现的任何错误都会被捕获，最后 storage provider 会执行 get 调用来确认该资源是否被成功创建。如果需要额外的清理工作 (finalization)，就会调用后期创建的处理器和装饰器；
1. 最后，[构造 HTTP 响应](https://github.com/kubernetes/apiserver/blob/kubernetes-1.14.0/pkg/endpoints/handlers/create.go#L170-L177)并返回给客户端。

这么多步骤！能够坚持走到这里是非常了不起的，并且我们意识到了 kube-apiserver 实际上做了很多工作。总结一下：我们部署的 Deployment 现在存在于 etcd 中，但仍没有看到它真正的 work...

**注**：在 Kubernetes v1.14 之前，这往后还有 Initializer 的步骤，该步骤在 v1.14 [被 webhook admission 取代](https://github.com/kubernetes/kubernetes/issues/67113)。

## Control loops

### Deployment Controller

截至目前，我们的 Deployment 已经存储于 etcd 中，并且所有的初始化逻辑都已完成。接下来的阶段将涉及 Deployment 所依赖的资源拓扑结构。

在 Kubernetes， Deployment 实际上只是 ReplicaSet 的集合，而 ReplicaSet 是 Pod 的集合。那么 Kubernetes 如何从一个 HTTP 请求创建这个层次结构呢？这就不得不提 Kubernetes 的内置控制器 （Controller）。

Kubernetes 系统中使用了大量的 Controller， Controller 是一个用于将系统状态从`当前状态`调谐到`期望状态`的异步脚本。所有内置的 Controller 都通过组件 kube-controller-manager 并行运行，每种 Controller 都负责一种具体的控制流程。

首先，我们介绍一下 Deployment Controller：

将 Deployment 存储到 etcd 后，我们通过 kube-apiserver 可以看到它。当这个新资源可用时， Deployment Controller 会检测到它，它的工作是监听 Deployment 的更改。在我们的例子中， Controller 通过[注册创建事件的回调函数](https://github.com/kubernetes/kubernetes/blob/v1.14.0//pkg/controller/deployment/deployment_controller.go#L122)（更多相关信息，参见下文）。

当我们的 Deployment 首次可用时，将执行此回调函数，并[将该对象添加到内部工作队列（internal work queue）](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/deployment/deployment_controller.go#L166-L170)。

当它处理我们的 Deployment 对象时，控制器将[检查我们的 Deployment](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/deployment/deployment_controller.go#L571) 并意识到没有与之关联的 ReplicaSet 或 Pod。

它通过使用标签选择器 (label selectors) 查询 kube-apiserver 来实现此功能。有趣的是，这个同步过程是状态不可知的。另外，它以相同的方式调谐新对象和已存在的对象。

在意识到没有与其关联的 ReplicaSet 或 Pod 后，Deployment Controller 就会开始执行[弹性伸缩流程 (scaling process)](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/deployment/sync.go#L378)。它通过推出（例如，创建）一个 ReplicaSet， 为其分配 label selector 并将其版本号设置为 1。

ReplicaSet 的 PodSpec 字段是从 Deployment 的 manifest 以及其他相关元数据中复制而来。有时 Deployment 在此之后也需要更新（例如，如果设置了 process deadline）。

当完成以上步骤之后，该 [Deployment 的 status 就会被更新](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/deployment/sync.go#L67)，然后重新进入与之前相同的循环，等待 Deployment 与期望的状态相匹配。由于 Deployment Controller 只关心 ReplicaSet， 因此调谐过程将由 ReplicaSet Controller 继续。

### ReplicaSet Controller

在上一步中，Deployment Controller 创建了属于该 Deployment 的第一个 ReplicaSet， 但仍然没有创建 Pod。 所以这里我们需要引入 ReplicaSet Controller！

ReplicaSet Controller 的工作是监视 ReplicaSet 及其相关资源 Pod 的生命周期。与大多数其它控制器一样，它通过触发某些事件的处理程序来实现。

当创建 ReplicaSet 时（由 Deployment Controller 创建），ReplicaSet Controller 会[检查新 ReplicaSet 的状态](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/replicaset/replica_set.go#L583)，并意识到现有状态与期望状态之间存在偏差。然后，它试图通过[调整 pod 的副本数](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/replicaset/replica_set.go#L460)来调谐这种状态。

Pod 的创建也是[批量](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/controller/replicaset/replica_set.go#L478-L499))进行的，从数量 `SlowStartInitialBatchSize` 开始，然后在每次成功的迭代中以一种 `slow start` 操作加倍。这样做的目的是在大量 Pod 启动失败时（例如，由于资源配额），可以减轻 kube-apiserver 由于大量不必要的 HTTP 请求导致崩溃的风险。

Kubernetes 通过 Owner References （子资源的某个字段中引用其父资源的 ID） 来执行严格的资源对象层级结构。这确保了一旦 Controller 管理的资源被删除（级联删除），子资源就会被垃圾收集器删除，同时还为父资源提供了一种有效的方式来避免他们竞争同一个子资源（想象两对父母认为他们拥有同一个孩子的场景）。

Owner References 的另一个好处是，它是有状态的。如果重启任何的 Controller，那么由于资源对象的拓扑关系与 Controller 无关，该重启时间不会影响到系统的稳定运行。这种对资源隔离的重视也体现在 Controller 本身的设计中： Controller 不能对自己没有明确拥有的资源进行操作，它们之间互不干涉，互不共享。

有时系统中也会出现孤儿 （orphaned） 资源，通常由以下两种途径产生：

- 父资源被删除，但子资源没有被删除
- 垃圾收集策略禁止删除子资源

当发生这种情况时， Controller 将会确保孤儿资源拥有新的 Owner。 多个父资源可以相互竞争同一个孤儿资源，但只有一个会成功（其他父资源会收到一个验证错误）。

### Informers

你可能已经注意到，有些 Controller（例如 RBAC 授权器或 Deployment Controller）需要检索集群状态然后才能正常运行。

以 RBAC authorizer 举例，当请求进入时， authorizer 会将用户的初始状态缓存下来供以后使用，然后用它来检索与 etcd 中的用户关联的所有`角色（Role）`和`角色绑定（RoleBinding）`。

那么 Controller 是如何访问和修改这些资源对象的呢？答案是引入 Informer。

Infomer 是一种模式，它允许 Controller 订阅存储事件并列出它们感兴趣的资源。除了提供一个很好的工作抽象，它还需要处理很多细节，如缓存（缓存很重要，因为它减少了不必要的 kube-apiserver 连接，并减少了服务器端和控制端的重复序列化成本）。通过使用这种设计，它还允许 Controller 以线程安全 （thread safe） 的方式进行交互，而不必担心线程冲突。

有关 Informer 的更多信息，可深入阅读 [《Kubernetes: Controllers, Informers, Reflectors and Stores》](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores)

### Scheduler

当所有的 Controller 正常运行后，etcd 中就会保存一个 Deployment、一个 ReplicaSet 和 三个 Pod， 并且可以通过 kube-apiserver 查看到。然而，这些 Pod 还处于 `Pending` 状态，因为它们还没有被调度到集群中合适的 Node 上。最终解决这个问题的 Controller 是 Scheduler。

Scheduler 作为一个独立的组件运行在集群控制平面上，工作方式与其他 Controller 相同：监听事件并调谐状态。

具体来说， Scheduler 的作用是过滤 PodSpec 中 `NodeName` 字段为空的 Pod 并尝试将其调度到合适的节点。

为了找到合适的节点， Scheduler 会使用特定的算法，默认调度算法工作流程如下：

1. 当 Scheduler 启动时，会注册[一系列默认的预选策略](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/scheduler/algorithmprovider/defaults/defaults.go#L37)，这些预选策略会[对候选节点进行评估](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/scheduler/core/generic_scheduler.go#L184)，判断候选节点是否满足候选 Pod 的需求。例如，如果 PodSpec 显式地限制了 CPU 和内存资源，并且节点的资源容量不满足候选 Pod 的需求时，Pod 就不会被调度到该节点上（资源容量 = 节点资源总量 - 节点中已运行的容器需求资源 （CPU 和内存）总和）；
1. 一旦选择了适当的节点，就会对剩余的节点运行一系列[优先级函数](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/scheduler/core/generic_scheduler.go#L639-L645)，以对候选节点进行打分。例如，为了在整个系统中分散工作负载，它将偏好于资源请求较少的节点（因为这表明运行的工作负载较少）。当它运行这些函数时，它为每个节点分配一个成绩。然后选择分数最高的节点进行调度。

一旦算法找到了合适的节点， Scheduler 就会[创建一个 Binding 对象](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/scheduler/scheduler.go#L559-L565)，该对象的 Name 和 Uid 与 Pod 相匹配，并且其 `ObjectReference` 字段包含所选节点的名称，然后通过[发送 POST 请求](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/scheduler/factory/factory.go#L734)给 kube-apiserver。

当 kube-apiserver 接收到此 Binding 对象时，注册表会将该对象反序列化 （registry deserializes） 并更新 Pod 资源中的以下字段：

1. [将 NodeName 的值设置为 Binding 对象 ObjectReference 中的 NodeName](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/registry/core/pod/storage/storage.go#L176)；
1. [添加相关的注释 (annotations)](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/registry/core/pod/storage/storage.go#L180-L182)；
1. [将 PodScheduled 的 status 设置为 True](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/registry/core/pod/storage/storage.go#L183-L186)。

一旦 Scheduler 将 Pod 调度到某个节点上，该节点的 Kubelet 就会接管该 Pod 并开始部署。

附注：自定义调度器：有趣的是预测和优先级函数 （predicates and priority functions） 都是可扩展的，可以使用参数 `--policy-config-file` 来定义。这引入了一定程度的灵活性。管理员还可以在独立部署中运行自定义调度器（具有自定义处理逻辑的调度器）。如果 PodSpec 中包含 `schedulerName`，Kubernetes 会将该 pod 的调度移交给使用该名称注册的调度器。

## Kubelet

### Pod Sync

截至目前，所有的 Controller 都完成了工作，让我们来总结一下：

1. HTTP 请求通过了认证、授权和准入控制阶段；
1. 一个 Deployment、ReplicaSet 和三个 Pod 被持久化到 etcd；
1. 最后每个 Pod 都被调度到合适的节点。

然而，到目前为止，所有的状态变化仅仅只是针对保存在 etcd 中的资源对象，接下来的步骤涉及到在工作节点之间运行具体的容器，这是分布式系统 Kubernetes 的关键因素。这些事情都是由 Kubelet 完成的。

在 Kubernetes 集群中，每个 Node 节点上都会启动一个 Kubelet 服务进程，该进程用于处理 Scheduler 下发到本节点的 Pod 并管理其生命周期。这意味着它将处理 Pod 与 Container Runtime 之间所有的转换逻辑，包括挂载卷、容器日志、垃圾回收等操作。

一个有用的方法，你可以把 Kubelet 当成一种特殊的 Controller，它每隔 20 秒（可以自定义）向 kube-apiserver 查询 Pod，过滤 NodeName 与自身所在节点匹配的 Pod 列表。

一旦获取到了这个列表，它就会通过与自己的内部缓存进行比较来检测差异，如果有差异，就开始同步 Pod 列表。我们来看看同步过程是什么样的：

1. 如果 Pod 正在创建， Kubelet 就会[暴露一些指标](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet.go#L1504)，可以用于在 Prometheus 中追踪 Pod 启动延时；
1. 然后，[生成一个 PodStatus 对象](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet_pods.go#L1333)，表示 Pod 当前阶段的状态。Pod 的 Phase 状态是 Pod 在其生命周期中的高度概括，包括 `Pending`，`Running`，`Succeeded`，`Failed` 和 `Unkown` 这几个值。状态的产生过程非常复杂，因此很有必要深入深挖一下：
    - 首先，串行执行一系列 `PodSyncHandlers`，每个处理器检查 Pod 是否应该运行在该节点上。当其中之一的处理器认为该 Pod 不应该运行在该节点上，则 Pod 的 Phase 值就会[变成 `PodFailed`](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet_pods.go#L1340-L1345) 并将从该节点被驱逐。例如，以 Job 为例，当一个 Pod 失败重试的时间超过了 `activeDeadlineSeconds` 设置的值，就会将该 Pod 从该节点驱逐出去；
    - 接下来，Pod 的 Phase 值由 init 容器和主容器状态共同决定。由于主容器尚未启动，容器被视为处于[等待阶段](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet_pods.go#L1284)，如果 [Pod 中至少有一个容器处于等待阶段，则其 Phase 值为 `Pending`](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet_pods.go#L1298-L1301)。、；
    - 最后，Pod 的 Condition 字段由 Pod 内所有容器状态决定。现在我们的容器还没有被容器运行时 (Container Runtime) 创建，所以，Kubelet [将 `PodReady` 的状态设置为 False](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/status/generate.go#L72-L83)。
1. 生成 PodStatus 之后，Kubelet 就会将它发送到 Pod 的 status 管理器，该管理器的任务是通过 kube-apiserver 异步更新 etcd 中的记录；
1. 接下来运行一系列 admit handlers 以确保该 Pod 具有正确的权限（包括强制执行 [AppArmor profiles 和 NO_NEW_PRIVS](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet.go#L864-L865)），在该阶段被拒绝的 Pod 将永久处于 `Pending` 状态；
1. 如果 Kubelet 启动时指定了 `--cgroups-per-qos` 参数，Kubelet 就会为该 Pod 创建 cgroup 并设置对应的资源限制。这是为了更好的 Pod 服务质量（QoS）；
1. 为 Pod [创建相应的数据目录](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kubelet_pods.go#L826-L839)，包括：
    - Pod 目录 (通常是 `/var/run/kubelet/pods/<podID>`)；
    - Pod 的挂载卷目录 (`<podDir>/volumes`)；
    - Pod 的插件目录 (`<podDir>/plugins`)。
1. 卷管理器会[挂载 `Spec.Volumes` 中定义的相关数据卷](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/volumemanager/volume_manager.go#L339)，然后等待挂载成功；
1. 从 kube-apiserver 中检索 `Spec.ImagePullSecrets`，然后将对应的 Secret 注入到容器中；
1. 最后，通过容器运行时 （Container Runtime） 启动容器（下面会详细描述）。

### CRI and pause container

到了这个阶段，大量的初始化工作都已经完成，容器已经准备好开始启动了，而容器是由容器运行时（例如 Docker）启动的。

为了更具可扩展性， Kubelet 使用 CRI （Container Runtime Interface） 来与具体的容器运行时进行交互。简而言之， CRI 提供了 Kubelet 和特定容器运行时实现之间的抽象。通过 [protocol buffers](https://github.com/google/protobuf)（一种更快的 JSON） 和 [gRPC API](https://grpc.io/)（一种非常适合执行 Kubernetes 操作的API）进行通信。

这是一个非常酷的想法，因为通过在 Kubelet 和容器运行时之间使用已定义的接口约定，容器编排的实际实现细节变得无关紧要。重要的是接口约定。这允许以最小的开销添加新的容器运行时，因为没有核心 Kubernetes 代码需要更改！

回到部署我们的容器，当一个 Pod 首次启动时， Kubelet [调用 RunPodSandbox 远程过程命令 （remote procedure command RPC）](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L65)。沙箱 （sandbox） 是描述一组容器的 CRI 术语，在 Kubernetes 中对应的是 Pod。这个术语是故意模糊的，因此其他不使用容器的运行时，不会失去其意义（想象一个基于 hypervisor 的运行时，沙箱可能指的是 VM）。

在我们的例子中，我们使用的是 Docker。 在 Docker 中，创建沙箱涉及创建 `pause` 容器。

`pause` 容器像 Pod 中的所有其他容器的父级一样，因为它承载了工作负载容器最终将使用的许多 Pod 级资源。这些“资源”是 Linux Namespaces (IPC，Network，PID)。

> 如果你不熟悉容器在 Linux 中的工作方式，那么我们快速回顾一下。 Linux 内核具有 Namespace 的概念，允许主机操作系统分割出一组专用资源（例如 CPU 或内存）并将其提供给一个进程，就好像它是世界上唯一使用它们的东西一样。 Cgroup 在这里也很重要，因为它们是 Linux 管理资源隔离的方式。 Docker 使用这两个内核功能来托管一个保证资源强制隔离的进程。更多信息，可深入阅读 [What even is a Container?](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

`pause` 容器提供了一种托管所有这些 Namespaces 的方法，并允许子容器共享它们。通过成为同一 Network Namespace 的一部分，一个好处是同一个 Pod 中的容器可以使用 localhost 相互访问。

`pause`  容器的第二个好处与 PID Namespace 有关。在这些 Namespace 中，进程形成一个分层树（hierarchical tree），顶部的“init” 进程负责“收获”僵尸进程。更多信息，请深入阅读 [great blog post](https://www.ianlewis.org/en/almighty-pause-container)。

创建 `pause` 容器后，将开始检查磁盘状态然后启动主容器。

### CNI and pod networking

现在，我们的 Pod 有了基本的骨架：一个 `pause` 容器，它托管所有 Namespaces 以允许 Pod 间通信。但容器网络如何运作以及建立的？

当 Kubelet 为 Pod 设置网络时，它将任务委托给 `CNI (Container Network Interface)` 插件。其运行方式与 Container Runtime Interface 类似。简而言之， CNI 是一种抽象，允许不同的网络提供商对容器使用不同的网络实现。

Kubelet 通过 stdin 将 JSON 数据（配置文件位于 `/etc/cni/net.d` 中）传输到相关的 CNI 二进制文件（位于 `/opt/cni/bin`） 中与之交互。下面是一个简单的示例 JSON 配置文件：

```yaml
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

CNI 插件还可以通过 `CNI_ARGS` 环境变量为 Pod 指定其他的元数据，包括 Pod Name 和 Namespace。

接下来会发生什么取决于 CNI 插件，这里，我们以 `bridge` CNI 插件为例：

1. 该插件首先会在 Root Network Namespace（也就是宿主机的 Network Namespace） 中设置本地 Linux 网桥，以便为该主机上的所有容器提供网络服务；
1. 然后它会将一个网络接口 （veth 设备对的一端）插入到 `pause` 容器的 Network Namespace 中，并将另一端连接到网桥上。你可以这样来理解 veth 设备对：它就像一根很长的管道，一端连接到容器，一端连接到 Root Network Namespace 中，允许数据包在中间传输；
1. 然后它会为 `pause` 容器的网络接口分配一个 IP 并设置相应的路由，于是 Pod 就有了自己的 IP。IP 的分配是由 JSON 配置文件中指定的 IPAM Plugin 实现的；
    - IPAM Plugin 的工作方式和 CNI 插件类似：通过二进制文件调用并具有标准化的接口，每一个 IPAM Plugin 都必须要确定容器网络接口的 IP、子网以及网关和路由，并将信息返回给 CNI 插件。最常见的 IPAM Plugin 称为 host-local，它从预定义的一组地址池为容器分配 IP 地址。它将相关信息保存在主机的文件系统中，从而确保了单个主机上每个容器 IP 地址的唯一性。
1. 对于 DNS， Kubelet 将为 CNI 插件指定 Kubernetes 集群内部 DNS 服务器 IP 地址，确保正确设置容器的 `resolv.conf` 文件。

### Inter-host networking

到目前为止，我们已经描述了容器如何与宿主机进行通信，但跨主机之间的容器如何通信呢？

通常情况下， Kubernetes 使用 Overlay 网络来进行跨主机容器通信，这是一种动态同步多个主机间路由的方法。一个较常用的 Overlay 网络插件是 `flannel`，它提供了跨节点的三层网络。

flannel 不会管容器与宿主机之间的通信（这是 CNI 插件的职责），但它对主机间的流量传输负责。为此，它为主机选择一个子网并将其注册到 etcd。然后，它保留集群路由的本地表示，并将传出的数据包封装在 UDP 数据报中，确保它到达正确的主机。

更多信息，请深入阅读 [CoreOS's documentation](https://github.com/coreos/flannel)。

### Container startup

所有的网络配置都已完成。还剩什么？真正地启动工作负载容器！

一旦 `sanbox` 完成初始化并处于 `active` 状态， Kubelet 将开始为其创建容器。首先[启动 PodSpec 中定义的 Init Container](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L736)，然后再启动主容器。具体过程如下：

1. [拉取容器的镜像](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L95)。如果是私有仓库的镜像，就会使用 PodSpec 中指定的 imagePullSecrets 来拉取该镜像；
1. [通过 CRI 创建容器](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L124)。 Kubelet 使用 PodSpec 中的信息填充了一个 `ContainerConfig` 数据结构（在其中定义了 command， image， labels， mounts， devices， environment variables 等），然后通过 protobufs 发送给 CRI。 对于 Docker 来说，它会将这些信息反序列化并填充到自己的配置信息中，然后再发送给 Dockerd 守护进程。在这个过程中，它会将一些元数据（例如容器类型，日志路径，sandbox ID 等）添加到容器中；
1. 然后 Kubelet 将容器注册到 CPU 管理器，它通过使用 `UpdateContainerResources` CRI 方法给容器分配给本地节点上的 CPU 资源；
1. 最后[容器真正地启动](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L144)；
1. 如果 Pod 中包含 [Container Lifecycle Hooks](https://v1-14.docs.kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)，容器启动之后就会[运行这些 Hooks](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubelet/kuberuntime/kuberuntime_container.go#L170-L185)。 Hook 的类型包括两种：Exec（执行一段命令） 和 HTTP（发送HTTP请求）。如果 PostStart Hook 启动的时间过长、挂起或者失败，容器将永远不会变成 Running 状态。

## Wrap-up

最后的最后，现在我们的集群上应该会运行三个容器，分布在一个或多个工作节点上。所有的网络，数据卷和秘钥都由 Kubelet 填充，并通过 CRI 接口添加到容器中并配置成功！
