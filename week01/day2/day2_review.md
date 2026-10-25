# 第二天学习成果评审

更新日期：2026-10-25。依据最新 [task_acceptance_day2.md](/Users/wty/workspace/study/week01/day2/task_acceptance_day2.md)，已重新查看 [Pod 图](/Users/wty/workspace/study/.images/week01/day2/task_acceptance_day2/image.png)，并核对本地 Kubernetes v1.32 系列源码。实际学习日期、用时未提供。

本文件保存第 2 天的最新 review；后续修订直接更新本文件，不按轮次新增文件。原始提交和图片未改动。

## 验收结论

**第二天部分通过，理解复验继续。** Into 的解码目标已补充，方向正确；当前文件的 CoreV1 调用对象和完整返回类型仍有旧表述。本次按用户要求讲解 cgroup 与已绑定 Pod 的 Pending，讲解不等同于理解已验证。**Pod 图修正和实际 Pod 字段采集任务已由用户取消，移出验收要求。**

| 项目 | 当前判断 | 依据 |
| --- | --- | --- |
| 配置加载源码 | 基本通过，签名需完整 | 已写出最终 *rest.Config；复盘回答仍漏指针和 error |
| 客户端及 List | 部分通过 | 已改为客户端集合，WatchList 描述正确；第 17 行调用对象仍错误 |
| Do/Into | 本次补充基本通过 | 已写明 decoder 读取 body 并写入传入 list；具体承载数据的是接口中的 PodList 指针 |
| UTS namespace | 本次实验通过 | 前后宿主 hostname/标识一致，内部不同，证据闭环成立 |
| cgroup | 观察部分通过，解释待补 | 已确认 v2、shell 归属和内存字段；cpu.max 缺项原因与字段解释未验证 |
| Pod 图与实际字段采集 | 用户取消 | 不再要求重画或提交实机字段，不计为完成或已验证；Pending 机制讲解继续 |
| 两道复盘回答 | 均部分正确 | 类型方向正确但理由不足；Pending 不矛盾答对，后续排障方向有误 |

## 1. 源码：把输入与返回类型写准确

### 配置接口与最终配置是两种东西

位置：[第 8～9 行](/Users/wty/workspace/study/week01/day2/task_acceptance_day2.md:8)。

这部分本次已经改正，不再列为原有错误。更精确地说，是指针类型 *DeferredLoadingClientConfig 满足接口。准确关系为：

```text
NewNonInteractiveDeferredLoadingClientConfig(...)
  → 声明返回类型：clientcmd.ClientConfig 接口
  → 其中的具体类型：*DeferredLoadingClientConfig

调用其 ClientConfig()
  → 返回 (*rest.Config, error)
```

第 3 步现已写出最终的 *rest.Config，正确。ClientConfig 既出现在接口名中，又出现在方法名中，需要继续保留这一区分。

核对入口：[merged_client_builder.go](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/tools/clientcmd/merged_client_builder.go:53)、[BuildConfigFromFlags](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/tools/clientcmd/client_config.go:665)。

### NewForConfig 创建客户端集合，不是控制器

位置：[第 13 行](/Users/wty/workspace/study/week01/day2/task_acceptance_day2.md:13)已改为客户端集合，正确。新增的 CoreV1 描述也已看到：它返回 c.coreV1 中的客户端，以 corev1.CoreV1Interface 接口类型返回；这里是核心 API 组的客户端，不是 Pod 等资源对象本身。第 17 行仍保留旧的 ClientConfig 表述，完整返回类型也尚未在复盘回答中写出。

```go
cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
// cfg：*rest.Config

clientset, err := kubernetes.NewForConfig(cfg)
// clientset：*kubernetes.Clientset
```

以上片段仅标明类型，实际程序仍需检查每次 error。

rest.Config 保存服务地址、认证/TLS、超时等配置；Clientset 提供访问 API 的入口。NewForConfig 内部会构建 HTTP client，但返回的是 Clientset。这里没有启动 informer、队列和协调逻辑。

**第 17 行仍有错误：**应写 `clientset.CoreV1().Pods("").List(...)`，而不是调用 ClientConfig 的 CoreV1。你的复盘回答已知道 Clientset 可以调用它，但正文仍未与回答保持一致。ClientConfig 接口负责加载配置，没有 CoreV1 方法。核对：[ClientConfig 接口](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/tools/clientcmd/client_config.go:60)、[CoreV1 方法](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/kubernetes/clientset.go:322)。

复盘回答的类型方向正确，但第 90 行遗漏指针和 error。完整返回类型分别是 `(*restclient.Config, error)`、`(*kubernetes.Clientset, error)`。restclient 是源码对 k8s.io/client-go/rest 的真实导入别名，**别名没有错**。核对：[BuildConfigFromFlags](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/tools/clientcmd/client_config.go:665)、[NewForConfig](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/kubernetes/clientset.go:484)。

第 91 行“名称里有 client，所以不是控制器”的理由不成立。控制器也会使用客户端。应根据行为判断：NewForConfig 只构造客户端，没有建立“观察实际状态 → 与期望比较 → 执行协调动作”的控制逻辑。单纯循环 List 并打印也不会自动修复资源。这题登记为：**类型识别部分通过，客户端与控制器的职责边界需巩固。**

### List 分支：本次已修正

位置：[第 19 行](/Users/wty/workspace/study/week01/day2/task_acceptance_day2.md:19)。你现在关于默认关闭、启用且条件适用才尝试 watch-list 的描述正确。

按本次 client-go v0.32.0 基线，WatchListClient 默认关闭。启用且条件适用时才尝试 watch-list；否则进入普通 list，尝试失败时也可回退。你贴出的 `.Do(ctx).Into(list)` 就在普通 list 分支。

外层 for 反复调用 List 是轮询；一次 List 返回 PodList 快照，即使通过 WatchList 完成快照，也不等于让调用者持续接收 Watch 事件。

已确认正确的部分：`Pods("")` 表示所有 namespace；List 接收 context 和 ListOptions，返回 `(*corev1.PodList, error)`，其中 Items 为 Pod 列表。跨 namespace 查询仍受授权约束。

核对入口：[gate 默认值](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/features/known_features.go:76)、[List 分支](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/gentype/type.go:174)。

### Do/Into：本次补充基本通过

本次新增的 decoder.Decode、读取 Result.body、写入传入 list 的说明方向正确。更精确地说，Into 的形参声明为 obj runtime.Object；在当前调用中，传入的 list 通常是 *corev1.PodList，解码修改它指向的对象。不是把响应字节直接存进一个抽象的“接口容器”。此处不再登记为缺少解码目标。

Do 执行请求并处理响应，返回 rest.Result；Into(list) 将响应解码写入已有的 list 对象，自身返回 error，不再发送请求。传输格式也不一定是 JSON，你的片段已经包含 Protobuf 偏好。

源码入口：[Do](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/rest/request.go:1269)、[Into](/Users/wty/workspace/study/k8s/kubernetes/staging/src/k8s.io/client-go/rest/request.go:1581)。

本地相关源码已与 v1.32.0 基线核对，所评审的 clientcmd、gentype、gate、watchlist、request 文件无差异。没有运行集群或构建测试，不把静态源码核对当作你的实机验证。

## 2. UTS：本次通过，不需要重复实验

第 43～53 行现在有完整对照：

| 位置 | hostname | UTS 标识 |
| --- | --- | --- |
| 执行前宿主 shell | k8s01 | uts:[4026531838] |
| unshare 内 | k8s-week1 | uts:[4026532732] |
| 退出后宿主 shell | k8s01 | uts:[4026531838] |

“外部 → 内部 → 返回外部”的证据闭环已经成立，支持本次 hostname 隔离的判断。验收范围限定于这项观察，不扩大到网络、PID 或文件系统隔离。[UTS 手册](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)

## 3. cgroup：已有观察证据，还缺解释与缺失项诊断

[第 65～79 行](/Users/wty/workspace/study/week01/day2/task_acceptance_day2.md:65)已经完成有效采集，不再标为“没有做实验”。

| 实际输出 | 可以得出的结论 | 还不能得出的结论 |
| --- | --- | --- |
| cgroup2fs 与 0::/user.slice/... | 当前 shell 记录了 v2 归属路径 | 不能认为已经找到了某个 Pod 的资源组 |
| memory.max: max | 当前组未设置自身 memory.max 数值上限 | 不等于无限可用内存；仍受祖先组和物理资源约束 |
| memory.current: 2715648 | 采样时计入该组及后代的内存为 2715648 字节，约 2.59 MiB | 不等于整机使用量，也不是只统计 shell 的 RSS |
| 没有 cpu.max 输出 | 当前记录未给出 CPU 配额 | 不能当作 max，也不能判定 CPU 故障 |

字段与层级含义依据[内核 cgroup v2 文档](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files)。以上是评审解释，仍需你用自己的话说明观察范围，才能验证理解。

### 先查明 cpu.max 为什么没有输出

上次示例使用 if -f 静默跳过不存在的文件，容易掩盖差异；这不是你执行失败。当前只能确认“输出缺项”，还不能认定具体原因。

在原 Linux 实验机的宿主 shell 只读执行，保留输出和报错。不同会话的 session 编号可能变化，因此重新取路径：

```bash
STUDY_CGROUP_PATH=$(awk -F: '$1 == "0" {print $3}' /proc/$$/cgroup)
STUDY_CGROUP_DIR="/sys/fs/cgroup${STUDY_CGROUP_PATH}"
STUDY_CGROUP_PARENT=$(dirname "$STUDY_CGROUP_DIR")

for STUDY_CGROUP_FILE in \
  "$STUDY_CGROUP_DIR/cpu.max" \
  "$STUDY_CGROUP_DIR/cgroup.controllers" \
  "$STUDY_CGROUP_PARENT/cgroup.subtree_control" \
  /sys/fs/cgroup/cgroup.controllers; do
    printf '\n%s\n' "$STUDY_CGROUP_FILE"
    cat "$STUDY_CGROUP_FILE"
done
```

父组的 cgroup.subtree_control 决定向子组启用哪些控制器；未启用 cpu 是缺少 CPU 控制文件的一种可能。对照实际输出判断，不预写成你的实验结论。[Linux 5.15 控制器启用规则](https://docs.kernel.org/5.15/admin-guide/cgroup-v2.html#enabling-and-disabling)

若确认当前组没有 cpu.max，记录原因或仍待确认的假设即可，不要求为验收修改系统配置。数值解释可以另用文档示例练习，但要标为练习，不冒充本机输出。

### 本次补充讲解：归属、用量、上限是三个不同问题

可以把每个 cgroup 目录理解为一个资源分组，里面组织了进程，并通过不同控制器统计、约束资源。你的路径表示 shell 在一个登录会话组中；memory.current 是这组及后代的计账用量，memory.max 是这一层的硬限制设置，两者不是“使用量”和“机器总容量”的关系。

下面只是教学假设：父组 memory.max=1 GiB，子组 memory.max=max。子组没有额外收紧限制，仍受父组约束；父组的 1 GiB 还是它整个子树共享的预算，不是分配给每个子组各 1 GiB。

控制器文件出现也依赖层级。假设 user-0.slice/cgroup.subtree_control 只有 memory，没有 cpu，则直接子组 session-542.scope 可以有 memory.* 而没有 cpu.max。这里要看**父组**的 subtree_control；当前组自己的 subtree_control 决定的是它的孩子。若父组 cgroup.controllers 中有 cpu 而 subtree_control 没有 cpu，就能解释该子组缺少 CPU 限制接口。若父组 controllers 也没有 cpu，再向祖先查；这仍不是“机器没有 CPU”。

cpu.max 是 CPU 时间预算。例如文档练习值 50000 100000 表示每个 100000 微秒周期、整个组最多合计使用 50000 微秒 CPU 时间，约为 0.5 个 CPU 的持续算力额度，不表示固定绑定半个核心。文件存在且为 max 100000，表示这一层没有设该配额；文件不存在则需先检查控制器与路径。两种情况不能混为一谈。[Linux 5.15 CPU 接口](https://docs.kernel.org/5.15/admin-guide/cgroup-v2.html#cpu-interface-files)

### 阅读仍按小段推进

建议本周把全文翻译改为“围绕一个问题阅读一小段，再用命令验证”。英文能力提升与主线实验可以合并，不需要先花一周翻完文档才能继续。

本次只读：

1. [Kubernetes v1.32 中文说明](https://v1-32.docs.kubernetes.io/zh-cn/docs/concepts/architecture/cgroups/)中的“什么是 cgroup v2”和“识别版本”。
2. [内核英文文档](https://docs.kernel.org/admin-guide/cgroup-v2.html)的 What is cgroup?、cpu.max、memory.max、memory.current，以及本次用到的控制器启用条目。

每次英文练习控制在 10～15 分钟、约 100～150 词以内；保留原文链接，摘关键词，用自己的话写中文含义，再对照一次实际输出。翻译不追求逐字对应，重点识别字段的单位、范围和条件。该时间计入原学习预算。

本次补验要求“说明归属 → 解释实际字段 → 记录缺项原因或待证假设”，不考整篇翻译。遇到环境差异先记录，不增加修改控制值或触发 OOM 的实验。

## 4. Pod 图与实际字段采集：按用户要求取消

2026-10-25 用户明确取消这两项任务。不再要求修正图片、重画草图或采集实际 Pod 字段，不作为第 2 天未通过的原因；原始提交和图片保留。不将取消记作完成，也不据此登记实机排障能力通过。

## 5. 对已绑定但 Pending 的回答：排障方向需纠正

第 93 行“不矛盾”答对了。已绑定的 Pod 可能仍在拉镜像、运行 init 容器、准备网络或挂载卷。此时应根据证据定位节点启动过程，不能直接推导为“又变得不可调度”。同一 UID 的 Pod 不会因这些问题重新进入普通调度队列；换节点运行通常涉及新建的替代 Pod。[v1.32 Pod lifetime](https://v1-32.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-lifetime)

仅凭 nodeName 也不能证明经过全部打分流程，例如可以直接指定 spec.nodeName。资源变化仍可能影响节点启动或导致驱逐，但要区分阶段和处理组件。

不能只写“看 Events”，要进一步明确看什么：容器 waiting.reason/message、init 容器状态、conditions，以及事件的时间、报告组件、reason/message。FailedScheduling、FailedMount、镜像拉取问题指向不同链路。

源码核对：[有 nodeName 即 assigned](/Users/wty/workspace/study/k8s/kubernetes/pkg/scheduler/eventhandlers.go:312)、[未调度队列过滤条件](/Users/wty/workspace/study/k8s/kubernetes/pkg/scheduler/eventhandlers.go:384)、[kubelet 对 Pending 的判断](/Users/wty/workspace/study/k8s/kubernetes/pkg/kubelet/kubelet_pods.go:1689)。本次只看这几小段，不展开整个调度器。

这题登记为：**状态组合判断通过，组件职责和后续采证需巩固。**

### 本次补充讲解：选节点与在节点上启动是两个阶段

以普通调度场景为例：scheduler 先为 Pod 选择并绑定 worker-1，随后该节点的 kubelet 协调存储、容器运行时、网络等依赖启动工作负载。绑定完成不等于这些启动工作已经完成；Pod phase 的 Pending 同时覆盖等待调度和部分启动准备时间。

假设镜像仓库凭据错误：节点已选好，但 kubelet 调用运行时拉镜像失败，容器还不能启动。nodeName=worker-1、PodScheduled=True、phase=Pending 完全可以同时成立。修复凭据和拉取链路才针对这个问题，不能把“还没启动”直接归因于调度失败。

| 场景示例 | 优先分析的链路 |
| --- | --- |
| 未绑定，FailedScheduling / Insufficient cpu | 调度器、requests、节点可分配量与放置约束 |
| 已绑定，ErrImagePull / ImagePullBackOff | kubelet、运行时、镜像地址、仓库访问与凭据 |
| 已绑定，FailedMount | kubelet 挂载流程、卷依赖、必要时存储插件 |
| 已绑定，FailedCreatePodSandBox 且消息指向网络 | kubelet、运行时与 CNI，结合详细消息继续定位 |

这些都是教学场景，没有宣称你的机器出现了对应故障。各准备动作并非全部严格串行；表格用于选择排查入口，不是所有 Pending 问题的穷举清单。

资源调度通常按 requests 做容量判断，不能保证节点实际用量始终不变，也不能保证镜像、网络、存储都可用。节点后来资源不足仍可能引起启动失败或驱逐；但同一个 UID 的 Pod 不会因此自动换个节点重新调度。若有相应工作负载控制器，原 Pod 被删除后可创建新的替代 Pod，再由调度器选择节点。[资源调度说明](https://v1-32.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-requests-are-scheduled)、[Pod lifetime](https://v1-32.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-lifetime)

当前状态为“已讲解，等待用户解释与迁移判断”，不把讲解记为掌握；不要求恢复已取消的实机采集任务。

## 6. 当前不足与最小补齐清单

当前需要提升的重点是：沿调用链记录准确类型，避免混淆配置与客户端；为 cgroup 输出补上单位、层级和推断边界；在概念上分清调度与节点启动阶段。Pod 图和字段采集已经取消，不再追补。

- [x] 修正最终 *rest.Config、客户端集合和 WatchList 分支说明。
- [x] 补 unshare 退出后的 hostname/UTS 标识，UTS 实验通过。
- [x] 提交当前 shell 的 cgroup 路径与内存字段观察。
- [x] 补充 Into 的解码写入目标，方向正确。
- [ ] 将正文残留的 ClientConfig.CoreV1 与复盘回答中的返回类型同步为正确写法；不要求重做源码实验。
- [ ] 为 cgroup 输出补中文解释，保留 cpu.max 缺项诊断结果，约 10～15 分钟；遇阻记录后停止。
- Pod 图修正和实际字段采集：用户取消，移出待办。
- [ ] 回答下面两道追问，再决定是否通过理解复验。

这些是原任务的补验，本周仍按 14 小时核心预算，不新增全文翻译、GPU 实验或完整调度器阅读任务。评审没有连接集群，不能将评审解释记为你已验证的实机能力。

## 7. 待续复盘：本轮两题

用户本次优先要求讲解，原控制器职责追问暂缓；本轮只围绕刚讲解的机制，题目均为假设，不要求实机证据。

1. 父组 memory.max=1 GiB，当前会话组 memory.max=max。仅凭当前组的 max，能否认为它可以持续使用 4 GiB 内存？请用自己的话解释父子组的关系。
2. 一个 Pod 已绑定 worker-1，仍 Pending，容器原因是 ErrImagePull。把 scheduler 打分规则改成“优先空闲节点”，能直接解决它吗？你会先查哪两类信息？

前两题的回答已在上文点评；本轮两题尚未回答，等你作答后继续，不预先登记通过。第 1 天原有待订正项保持原状态。
