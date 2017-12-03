---
approvers:
- mikedanese
title: 配置 kubelet 垃圾收集
---



* TOC
{:toc}



kubelet 的垃圾收集是非常有用的功能，它可以清除未使用的容器和镜像。kubelet 在每分钟和每五分钟分别回收容器和镜像。

不建议使用第三方的垃圾收集工具，因为这些工具可能会移除期望存在的容器进而破坏 kubelet 的行为。



### 镜像收集

Kubernetes 通过 imageManager 与 cadvisor 协作的方式管理所有镜像的生命周期。

收集垃圾镜像的策略考虑两个因素：
`HighThresholdPercent` 和 `LowThresholdPercent`。磁盘使用率超过高阈值将触发垃圾收集策略。该策略将删除最近最少使用的镜像直至满足低阈值。


### 容器收集

容器收集策略考虑三个用户自定义变量。`MinAge` 是容器可以被收集的最小运行时间。`MaxPerPodContainer` 是每个pod (UID, container name) 中允许拥有死亡容器的最大数。`MaxContainers`全局死亡容器的最大数。通过将 `MinAge` 设置为零并将 `MaxPerPodContainer` 和 `MaxContainers` 分别设置为小于零，可以单独禁用这些变量。



Kubelet作用于未能被识别的，被删除的或超出上述变量边界的容器。最久远的容器首先被移除。当每个 pod(`MaxPerPodContainer`) 允许的最大容器数超出全局死亡容器的界限(`MaxContainers`) 时，`MaxPerPodContainer` 和 `MaxContainer` 可能会相互冲突。`MaxPerPodContainer` 可以在根据以下情形进行调整：最坏的情况是将 `MaxPerPodContainer` 降级至1并排除最旧的容器。此外，已被删除的 pod 所拥有的容器一旦比`MinAge`更旧，也会被移除。



### 用户配置

用户可以通过以下 kubelet 参数调整镜像收集机制的阈值：

1. `image-gc-high-threshold`,触发镜像垃圾收集操作的磁盘使用百分比。
默认是90%
2. `image-gc-low-threshold`,镜像垃圾收集尝试将磁盘使用率释放至某百分比。
默认是80%



我们还允许用户通过以下 kubelet 参数来定制垃圾收集策略：
1. `minimum-container-ttl-duration`,已结束容器在被收集前的最小年龄。默认是0分钟，意味每个结束的容器都将被收集。
2. `maximum-dead-containers-per-container`, 每个容器要保留的旧实例的最大数量。默认是1。
3. `maximum-dead-containers`，全局保留的容器旧实例的最大数量。
默认是-1，即没有全局的限制。

容器可能在其有效期到期之前被收集。这些容器包含有助于故障排除的日志和其他数据。强烈推荐 `maximum-dead-containers-per-container` 有足够大的值，以便每个期望的容器至少保留1个死容器。类似地，也推荐为 `maximum-dead-containers` 设置较大值。[参考此 issue](https://github.com/kubernetes/kubernetes/issues/13287) 获取更多细节。



### 新旧更迭

这篇文档中介绍的一些 kubelet 垃圾收集功能未来会被 kubelet 驱逐替代。

包括：

| 当前标识 | 新的标识 | 解释 |
| ------------- | -------- | --------- |
| `--image-gc-high-threshold` | `--eviction-hard` or `--eviction-soft` | 新的驱逐信号可以触发镜像垃圾收集 |
| `--image-gc-low-threshold` | `--eviction-minimum-reclaim` | 驱逐回收可以完成同样的行为 |
| `--maximum-dead-containers` | | 一旦旧的日志存储在容器上下文之外，就不建议使用 |
| `--maximum-dead-containers-per-container` | | 一旦旧的日志存储在容器上下文之外，就不建议使用 |
| `--minimum-container-ttl-duration` | | 一旦旧的日志存储在容器上下文之外，就不建议使用 |
| `--low-diskspace-threshold-mb` | `--eviction-hard` or `eviction-soft` | 驱逐将磁盘阈值推广普及到其他资源 |
| `--outofdisk-transition-frequency` | `--eviction-pressure-transition-period` | 驱逐将磁盘压力推广普及到其他资源 |



见 [配置资源处理](/docs/concepts/cluster-administration/out-of-resource/) 获取更多细节。
