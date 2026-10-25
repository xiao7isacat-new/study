# Kubernetes API 概念

Kubernetes API 是一种通过http提供的基于资源（restful）的程序化接口。他支持通过http 动作（post put patch delete get）来检索，创建，更新和删除主要资源

对于一些资源，api 提供了额外的允许更细力度的子资源（例如pod 的详细信息和日志检索的单独视图）并且以不同的表现形式提供这些资源，旨在带来便利和提升效率

Kubernetes 支持通过“监视”（watch）机制，对资源变更进行高效通知：

在 Kubernetes API 中，“监视”（watch）是一个动词，用于以数据流的形式追踪 Kubernetes 中某个对象所发生的变化。它能够高效地检测到各类变更。

此外，Kubernetes 还提供了一致的“列表”（list）操作，使 API 客户端能够有效地缓存、追踪并同步资源的状态。
您可以在线查阅 API 参考文档，或继续阅读下文，以全面了解该 API。
