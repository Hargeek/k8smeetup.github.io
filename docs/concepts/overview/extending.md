---
title: 扩展 Kubernetes 集群
approvers:
- erictune
- lavalamp
- cheftako
- chenopis
cn-approvers:
- linyouchong
---


{% capture overview %}


Kubernetes 是高度可配置和可扩展的。因此，极少需要分发或提交补丁代码给 Kubernetes 项目。


本文档介绍自定义 Kubernetes 群集的选项。 本文档的目标读者是希望了解如何使 Kubernetes 集群满足其业务环境需求的 {% glossary_tooltip text="集群运维人员" term_id="cluster-operator" %} 。 Kubernetes 项目的贡献者或潜在的开发人员也可以从本文找到有用的信息，如对已存在扩展点和模式的介绍，以及它们的权衡和限制。

{% endcapture %}


{% capture body %}


## 概述


定制方法可以大致分为 *配置* 和 *扩展*。*配置* 只涉及更改标志参数，本地配置文件或 API 资源; *扩展* 涉及运行额外的程序或服务。本文档主要内容是关于扩展。


## 配置


关于 *配置文件* 和 *标志* 的说明文档位于在线文档的参考部分，按照二进制组件各自描述：

* [kubelet](/docs/admin/kubelet/)
* [kube-apiserver](/docs/admin/kube-apiserver/)
* [kube-controller-manager](/docs/admin/kube-controller-manager/)
* [kube-scheduler](/docs/admin/kube-scheduler/).


在托管的 Kubernetes 服务或受控安装的 Kubernetes 版本中，标志和配置文件可能并不总是可以更改的。 而且当它们可以进行更改时，它们通常只能由集群管理员进行更改。 此外，标志和配置文件在未来的 Kubernetes 版本中可能会发生变化，并且更改设置后它们可能需要重新启动进程。 出于这些原因，只有在没有其他选择的情况下才使用它们。


*内置策略 API*，例如 [ResourceQuota](/docs/concepts/policy/resource-quotas/)、[PodSecurityPolicies](/docs/concepts/policy/pod-security-policy/)、[NetworkPolicy](/docs/concepts/services-networking/network-policies/) 和 基于角色的权限控制 ([RBAC](/docs/admin/authorization/rbac/)), 是内置的 Kubernetes API。API 通常与托管的 Kubernetes 服务和受控的 Kubernetes 安装一起使用。 它们是声明性的，并使用与其他 Kubernetes 资源（如 Pod ）相同的约定，所以新的群集配置可以重复使用，并以与应用程序相同的方式进行管理。而且，当他们变稳定后，他们和其他Kubernetes API 一样享受 [定义支持政策](/docs/reference/deprecation-policy/)。 出于这些原因，在合适的情况下它们优先于 *配置文件* 和 *标志* 被使用。


## 扩展程序


扩展程序是指对 Kubernetes 进行扩展和深度集成的软件组件。它们适合用于支持新的类型和新型硬件。


大多数集群管理员会使用托管的或统一分发的 Kubernetes 实例。 因此，大多数 Kubernetes 用户需要
安装扩展程序，而且还有少部分用户甚至需要编写新的扩展程序。


## 扩展模式


Kubernetes 的设计是通过编写客户端程序来实现自动化的。任何 读和（或）写 Kubernetes API 的程序都可以提供有用的自动化工作。*自动化*程序可以运行在集群之中或之外。按照本文档的指导，您可以编写出高可用的和健壮的自动化程序。自动化程序通常适用于任何 Kubernetes 群集，包括托管群集和受管理安装的集群。


*控制器* 模式是编写适合 Kubernetes 的客户端程序的一种特定模式。 控制器通常读取一个对象的 `.spec` 字段，可能做出一些处理，然后更新对象的 `.status` 字段。


一个控制器是 Kubernetes 的一个客户端。 而当 Kubernetes 作为客户端调用远程服务时，它被称为 *Webhook* ，远程服务称为 *Webhook 后端*。 和控制器类似，Webhooks 增加了一个失败点。


在 webhook 模型里，Kubernetes 向远程服务发送一个网络请求。
在 *二进制插件* 模型里，Kubernetes 执行一个二进制（程序）。
二进制插件被 kubelet（如 [Flex 卷插件](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)
和 [网络插件](/docs/concepts/cluster-administration/network-plugins/)) 和 kubectl 所使用。


下图显示了扩展点如何与 Kubernetes 控制平面进行交互。

<img src="https://docs.google.com/drawings/d/e/2PACX-1vQBRWyXLVUlQPlp7BvxvV9S1mxyXSM6rAc_cbLANvKlu6kCCf-kGTporTMIeG5GZtUdxXz1xowN7RmL/pub?w=960&h=720">





## 扩展点


下图显示了 Kubernetes 系统的扩展点。

<img src="https://docs.google.com/drawings/d/e/2PACX-1vSH5ZWUO2jH9f34YHenhnCd14baEb4vT-pzfxeFC7NzdNqRDgdz4DDAVqArtH4onOGqh0bhwMX0zGBb/pub?w=425&h=809">




1.   用户通常使用 `kubectl` 与 Kubernetes API 进行交互。[kubectl 插件](docs/tasks/extend-kubectl/kubectl-plugins) 扩展了 kubectl 二进制程序。它们只影响个人用户的本地环境，因此不能执行站点范围的策略。

2.   apiserver 处理所有请求。 apiserver 中的几种类型的扩展点允许对请求进行身份认证或根据其内容对其进行阻止、编辑内容以及处理删除操作。这些内容在 [API 访问扩展](docs/concepts/overview/extending#api-访问扩展) 小节中描述。

3.   apiserver 提供各种 *资源*。 *内置的资源种类*，如 `pods` ，由 Kubernetes 项目定义，不能更改。 您还可以添加您自己定义的资源或其他项目已定义的资源，称为 *自定义资源*，如 [自定义资源](docs/concepts/overview/extending#用户自定义类型) 部分所述。 自定义资源通常与 API 访问扩展一起使用。

4.   Kubernetes 调度器决定将 pod 放置到哪个节点。 有几种方法可以扩展调度器。这些内容在 [Scheduler Extensions](docs/concepts/overview/extending#调度器扩展) 小节中描述。

5.   Kubernetes 的大部分行为都是由称为控制器的程序实现的，这些程序是 API-Server 的客户端。 控制器通常与自定义资源一起使用。

6.   kubelet 在主机上运行，并帮助 pod 看起来就像在集群网络上拥有自己的 IP 的虚拟服务器。[网络插件](docs/concepts/overview/extending#网络插件) 让您可以实现不同的 pod 网络。

7.  kubelet 也挂载和卸载容器的卷。 新的存储类型可以通过 [存储插件](docs/concepts/overview/extending#存储插件) 支持。


如果您不确定从哪里开始扩展，此流程图可以提供帮助。 请注意，某些解决方案可能涉及多种类型的扩展。


<img src="https://docs.google.com/drawings/d/e/2PACX-1vRWXNNIVWFDqzDY0CsKZJY3AR8sDeFDXItdc5awYxVH8s0OLherMlEPVUpxPIB1CSUu7GPk7B2fEnzM/pub?w=1440&h=1080">




## API 扩展

### 用户自定义类型


如果您想定义新的控制器、应用程序配置对象或其他声明式 API，并使用 Kubernetes 工具（如 `kubectl` ）管理它们，请考虑为 Kubernetes 添加一个自定义资源。


不要使用自定义资源作为应用、用户或者监控数据的数据存储。


有关自定义资源的更多信息，请查看 [自定义资源概念指南](/docs/concepts/api-extension/custom-resources.md)。



### 将新的 API 与自动化相结合


通常，在您添加新的 API 时，还需要添加一个读取和（或）写入新 API 的循环控制线程。 当使用自定义 API 和循环控制线程的组合来管理特定的（通常是有状态的）应用程序时，这称为 *Operator* 模式。 自定义 API 和循环控制线程也可以用来控制其他资源，例如存储、策略等等。


### 改变内置资源


当您通过添加自定义资源来扩展 Kubernetes API 时，添加的资源始终属于新的 API 组。 您不能替换或更改已有的 API 组。
添加 API 不会直接影响现有 API（例如 Pod ）的行为，但是 API 访问扩展可以。



### API 访问扩展


当请求到达 Kubernetes API Server 时，它首先被要求进行用户认证，然后要进行授权检查，接着受到各种类型的准入控制的检查。 有关此流程的更多信息，请参阅 [访问 API](/docs/admin/accessing-the-api/)]。


上述每个步骤都提供了扩展点。


Kubernetes 有几个它支持的内置认证方法。 它还可以位于身份验证代理之后，并将授权 header 中的 token 发送给远程服务进行验证（webhook）。 所有这些方法都在[身份验证文档](/docs/admin/authentication/) 中介绍。


### 身份认证


[身份认证](/docs/admin/authentication) 将所有请求中的 header 或证书映射为发出请求的客户端的用户名。


Kubernetes 提供了几种内置的身份认证方法，如果这些方法不符合您的需求，可以使用 [身份认证 webhook](/docs/admin/authentication/#webhook-token-authentication) 方法。



### 授权


[授权](/docs/admin/authorization/webhook/) 决定特定用户是否可以对 API 资源执行读取、写入以及其他操作。 它只是在整个资源的层面上工作 - 它不基于任意的对象字段进行区分。 如果内置授权选项不能满足您的需求，[授权 webhook](/docs/admin/authorization/webhook/) 允许调用用户提供的代码来作出授权决定。


 
### 动态准入控制


在请求被授权之后，如果是写入操作，它还将进入 [准入控制](/docs/admin/admission-controllers/) 步骤。 除了内置的步骤之外，还有几个扩展：


*   [镜像策略 webhook](/docs/admin/admission-controllers/#imagepolicywebhook) 限制了哪些镜像可以在容器中运行。

*   为了进行灵活的准入控制决策，可以使用通用的 [Admission webhook](/docs/admin/extensible-admission-controllers/#external-admission-webhooks)。 Admission Webhook 可以拒绝创建或更新操作。

*   [初始器](/docs/admin/extensible-admission-controllers/#initializers) 是可以在创建对象之前修改对象的控制器。初始器可以修改初始对象的创建，但不能影响对对象的更新。初始器也可以拒绝对象。


## 基础设施扩展



### 存储插件


[Flex Volumes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/flexvolume-deployment.md
) 允许用户挂载无内置插件支持的卷类型，它通过 Kubelet 调用一个二进制插件来挂载卷。



### 设备插件


设备插件允许节点通过 [设备插件](/docs/concepts/cluster-administration/device-plugins/) 发现新的节点资源（除了内置的 CPU 和内存之外）。



### 网络插件


不同的网络结构可以通过节点级的 [网络插件](/docs/admin/network-plugins/) 支持。


### 调度器扩展


调度器是一种特殊类型的控制器，用于监视 pod 并将其分配到节点。默认的调度器可以完全被替换，而
继续使用其他 Kubernetes 组件，或者可以同时运行 [多个
调度器](/docs/tasks/administer-cluster/configure-multiple-schedulers/)。


这是一个重要的任务，几乎所有的 Kubernetes 用户都发现他们不需要修改调度器。


调度器也支持 [webhook](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md) ，它允许一个 webhook 后端（调度器扩展程序）为 pod 筛选节点和确定节点的优先级。

{% endcapture %}


{% capture whatsnext %}


* 详细了解 [自定义资源](/docs/concepts/api-extension/custom-resources/)

* 了解 [动态准入控制](/docs/admin/extensible-admission-controller)

* 详细了解基础设施扩展
  * [网络插件](/docs/concepts/cluster-administration/network-plugin)
  * [设备插件](/docs/concepts/cluster-administration/device-plugins.md)

* 了解 [kubectl 插件](/docs/tasks/extend-kubectl/kubectl-plugin)

* 查看自动化的例子
  * [Operator 列表](https://github.com/coreos/awesome-kubernetes-extensions)

{% endcapture %}

{% include templates/concept.md %}


