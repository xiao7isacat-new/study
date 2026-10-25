# K8s 源码与 GPU 运维：四个月学习计划

制定日期：2026-10-20。按 16 个学习周执行，四个月内其余日历时间用于值班、等待实验机和补做验收。每周稳定投入 14 小时，额外 4 小时属于自愿进阶时间。

## 1. 根据你的情况确定目标

你有 3 年私有云运维经验，维护 3 套 1500～3000 节点的平台，接触 K8s 3 年、GPU 半年，做过 1.5 年运维开发。Go 处于入门阶段，namespace、cgroup 和容器网络需要从概念理解推进到实验和排障。

因此，学习重点放在“从生产现象追到实现、用实验验证判断、把经验写成工具”。K8s 基本命令只在发现缺口时补充。

四个月的目标按可验证能力定义：

| 方向 | 结业时应能独立完成 | 验收证据 |
| --- | --- | --- |
| K8s 源码 | 追踪控制器与 GPU Pod 两条主链；将 Pending、状态不收敛、容器启动失败等问题定位到实现层 | 两张带版本和符号的调用图、至少 4 份源码排障案例、一次陌生问题答辩 |
| GPU 运维 | 新机能力识别、分层验收、基线采集；区分应用、驱动、运行环境、PCIe/网络及疑似硬件问题 | 实机验收报告、结构化采集工具、至少 6 张故障案例卡 |
| Go 工程 | 用 client-go 实现可恢复、可观测、可部署的控制器 | 重复事件、冲突、重启、API 中断等异常用例通过；有真实 API Server 集成记录 |
| 综合项目 | GPU 节点健康管理 + 整卡 Job 调度编排，具备小范围试点条件 | 部署包、运行手册、回退方案、实机集成与负载报告 |

这是一份在控制器、调度与 GPU 节点链路形成深度的计划。四个月后的成熟度由证据决定；小集群通过与 3000 节点生产长期稳定运行需要分别记录。

## 2. 时间预算与调整规则

| 用途 | 每周必做 | 16 周合计 |
| --- | ---: | ---: |
| K8s 源码、Go 与容器机制，含每日英文练习 | 5 小时 | 80 小时 |
| GPU 文档、证据分析和实机实验 | 3 小时 | 48 小时 |
| 综合项目开发与验证 | 4 小时 | 64 小时 |
| 复盘、值班缓冲和补漏 | 2 小时 | 32 小时 |
| 合计 | 14 小时 | 224 小时 |

建议每周安排：工作日三次各 2 小时，分别用于源码、源码实验、GPU 分析；周末合计 8 小时，其中源码 1 小时、GPU 1 小时、项目 4 小时、复盘与缓冲 2 小时。GPU 机器突然可用时，用一次 3～4 小时窗口替换当周对应任务，准备和分析时间在邻近两周调剂。

额外 4 小时优先用于：补齐 Go 短板 → 把当周实验做完整 → Scheduler Framework 实践 → 有条件的 RoCE/NCCL 深入。额外时间不用于增加第二套平台或堆功能。

每周只认三类产出：一个解释清楚的机制、一个能复现的实验、一项项目增量。同一次实验可以同时支持源码和 GPU 学习，不重复计时。连续两周必做任务完成不足 70%，先减少范围；月考没通过，下一周先补核心缺口。

## 3. 实验环境和版本

### CPU 环境

准备一个可重建的 CPU 实验集群，建议 1 个控制平面和 2 个工作节点；另留一台 Linux 开发机。优先复用已有实验集群。若需从零搭建，将其计入第 1 周项目 4 小时，CLI 完整功能顺延至第 2 周，不额外占用你的时间。硬件按手头资源分配，不要求在学习开始时模拟数千个真实节点。可使用 Debian、containerd；先记录 Debian 主版本、内核及运行时版本。namespace、cgroup、CRI 和 CNI 实验在 Linux 节点完成。

源码阅读固定 Kubernetes `v1.32.0` 和 client-go `v0.32.0`，形成稳定教材；实验记录实际 `v1.32.x` patch。`client-go`、`api`、`apimachinery` 的依赖版本保持一致，工具链依据所选版本的 `go.mod`。源码中的函数和路径以固定 tag 为准。[client-go 版本说明](https://github.com/kubernetes/client-go/tree/v0.32.0)

v1.32 已于 2026-02-28 结束上游支持；它适合作为你指定的源码阅读基线。生产试点前另做目标集群和组件兼容性验证，并将支持状态列入发布决策。[Kubernetes 发布记录](https://kubernetes.io/releases/patch-releases/)

### GPU 环境

每次只持有 3～4 小时，优先手动打通 driver → Container Toolkit/containerd → NVIDIA Device Plugin → GPU Pod，先能解释每层，再学习 GPU Operator。Debian 不能直接套用 GPU Operator 的已验证平台结论，具体版本组合需查矩阵。[GPU Operator 平台矩阵](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)、[Toolkit 安装与平台](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)、[Toolkit 平台矩阵](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/supported-platforms.html)

每台机器建立能力档案：完整 GPU 型号与 edition、UUID、PCI ID、vBIOS、驱动、内核、GPU/NIC 拓扑、NIC 型号/固件、RoCE 配置。PRO6000 的代际和版本未确认前，不预设 MIG、ECC、DCGM 或 GPUDirect RDMA 的覆盖范围。对不支持的项目输出 `UNSUPPORTED`；缺少证据输出 `INCONCLUSIVE` 或 `NOT_TESTED`，不要写成通过。

## 4. 16 周任务

下表全部为每周 14 小时主线。带“实机窗口”的任务可前后移动；CPU 开发、日志分析和报告不随之停摆。

### 第 1 月：从使用 K8s 走到控制循环

| 周 | 源码与机制重点 | GPU 重点 | 项目增量和当周验收 |
| --- | --- | --- | --- |
| 1 | 固定版本、跑一个目标包测试；Go 的 interface、error、context；用进程和容器观察 namespace/cgroup | 认清 GPU/驱动/CUDA/runtime 的关系；准备机器档案 | 写只读 Node/Pod 清单 CLI，支持 JSON 和超时；提交环境清单、一个容器隔离实验。第 2 天 Pod 图修正与字段采集已于 2026-10-25 按用户要求取消 |
| 2 | RESTClient/Clientset、List/Watch、resourceVersion；理解断连后的重新同步 | 准备 driver/runtime 兼容检查；争取实机窗口 A | CLI 增加 watch 和退出处理；在真实实验 API Server 上验证断连恢复后状态正确；记录原始事件 |
| 3 | Reflector → DeltaFIFO → SharedInformer → Indexer/Lister；缓存同步与共享对象 | 裸机、容器、Pod 的 GPU 可见性分层；无卡时分析窗口 A 证据 | 改为 informer 观察器；按节点索引工作负载；能解释缓存滞后、DeepCopy 和初始同步 |
| 4 | typed workqueue、去重、限速重试、幂等协调、优雅退出；读 sample-controller | 整理 2 个“宿主机有卡但容器不可用”的案例 | 将目标 Node 的摘要协调到测试命名空间内自有 ConfigMap；验证重复事件、删除 ConfigMap 后重建、重启、临时失败后的收敛；完成第 1 月考核 |

### 第 2 月：打通调度、节点和 GPU 设备链路

| 周 | 源码与机制重点 | GPU 重点 | 项目增量和当周验收 |
| --- | --- | --- | --- |
| 5 | ReplicaSet 控制器、owner reference、expectations；API 更新冲突和 status | PCIe 代际/宽度、NUMA、GPU–NIC 拓扑；建立采集项 | 复现 Pod 删除后补建；画完整协调链；报告采集器 v0.1 能保留原始证据和命令退出码 |
| 6 | Scheduler 主循环、队列、snapshot、Filter/Score、assume/bind；保留默认约束 | 解释资源发布、请求和可分配量；争取实机窗口 B | 复现 GPU 数量不足、标签不符、污点 3 类 Pending；各给出事件、插件/符号与修复验证 |
| 7 | kubelet Pod workers → syncPod → kuberuntime → CRI；结合 CNI、cgroup 观察失败 | 温度、功耗、频率、利用率、显存；区分应用 OOM 与主机 OOM | 对 ImagePull、Sandbox/CNI、OOM 三类问题建立分层证据；项目输出明确失败原因 |
| 8 | Device Manager：注册、ListAndWatch、Allocate、健康变化、checkpoint、PodResources | Xid、ECC/DCGM 适用范围；争取实机窗口 C | 画 GPU Pod 从 API 到具体设备可用的链路；完成一份真实 GPU 验收报告与第 2 月考核 |

### 第 3 月：把脚本变成工程化控制器

| 周 | 源码与机制重点 | GPU 重点 | 项目增量和当周验收 |
| --- | --- | --- | --- |
| 9 | API 设计：spec/status、Conditions、observedGeneration；阅读错误处理与状态转换 | 按 GPU UUID/Node/Pod 关联指标和事件；缺失数据的含义 | 先实现 GpuNodeReport 对象及健康结果导入；状态包含采样时间、能力、证据引用和原因 |
| 10 | 按 sample-controller 实现 GPUWorkload → batch Job；默认调度器的资源与亲和性策略 | GPU Job 正确性、容器库/驱动不匹配、任务超时 | 能提交整卡任务、选型号/节点池、查看状态和失败原因；重复事件不重复创建 Job |
| 11 | 乐观并发、Patch、owner reference、退避、超时与取消；对照原生控制器 | 新机验收输出结构化 JSON 与可读报告；保留诊断覆盖范围 | 验证 409、任务被删、API 断连和控制器重启；修复后补针对性测试/复现脚本 |
| 12 | Lease leader election、最小 RBAC、配置、日志与指标；两副本恢复 | 健康状态去抖、过期、恢复复验；新异常如何处置 | 形成 v0.1；完成双副本切换、异常收敛测试和第 3 月考核 |

### 第 4 月：做成可交付、可解释的工具

| 周 | 源码与机制重点 | GPU 重点 | 项目增量和当周验收 |
| --- | --- | --- | --- |
| 13 | 健康状态与任务准入联动；定义写入者、字段归属、过期策略和竞争窗口 | 健康/疑似异常/隔离/恢复/未知的区别；人工确认处置 | 新任务避开不合格节点；旧任务有独立处置策略；演示健康结果过期和恢复复验 |
| 14 | 大规模设计：watch 范围、索引、限流、队列、pprof；避免每个事件全量扫描 | RoCE 分层：网卡/链路/GID/MTU、主机内存 RDMA；无双机时用日志包练习 | 逐步增加模拟对象负载，记录 API 请求、内存和队列时延；交一份瓶颈分析及一次改进对照 |
| 15 | API/控制器/插件异常的综合演练；发布、暂停、回退 | 争取实机窗口 D 做项目集成回归；NCCL 深入作为条件项 | 形成候选版本，完成 8 个核心异常场景；由另一人按文档部署并运行一次任务 |
| 16 | 源码答辩、陌生故障定位、范围与未验项复核 | 对新机器/完整证据包独立下结论，列复验条件 | 交部署包、验收工具、调用图、故障手册、测试报告和试点方案；完成最终考核 |

## 5. 源码入口和设计问题

每次按“一个现象 → 一个问题 → 一条调用链 → 一个实验 → 一份证据”阅读。先跟正常路径，再看失败和恢复；用调用关系扩展阅读范围。

| 主线 | 固定版本入口 | 阅读时必须回答的设计问题 |
| --- | --- | --- |
| List/Watch 与缓存 | [reflector.go](https://github.com/kubernetes/client-go/blob/v0.32.0/tools/cache/reflector.go)、[shared_informer.go](https://github.com/kubernetes/client-go/blob/v0.32.0/tools/cache/shared_informer.go) | 为什么使用 List/Watch？断连如何重新同步？缓存何时可用？事件通知和最终状态有什么区别？ |
| 工作队列 | [queue.go](https://github.com/kubernetes/client-go/blob/v0.32.0/util/workqueue/queue.go)、[rate_limiting_queue.go](https://github.com/kubernetes/client-go/blob/v0.32.0/util/workqueue/rate_limiting_queue.go)、[官方控制器示例](https://github.com/kubernetes/sample-controller/tree/v0.32.0) | dirty/processing 如何协作？何时 Done/Forget/重试？同一对象反复变化时为什么还能收敛？ |
| 原生控制器 | [replica_set.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/pkg/controller/replicaset/replica_set.go) | 缓存滞后为何可能造成重复创建？expectations 解决什么？接口和依赖如何支持测试？ |
| 调度 | [schedule_one.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/pkg/scheduler/schedule_one.go) | ScheduleOne 如何进入调度和绑定？assume 有什么作用？失败如何回到队列？插件边界在哪里？ |
| 节点与运行时 | [kubelet.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/pkg/kubelet/kubelet.go)、[pod_workers.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/pkg/kubelet/pod_workers.go) | 一个 Pod 如何串行协调？删除、重试、状态上报怎样交互？CRI 和 CNI 错误在哪里表现？ |
| GPU 设备分配 | [Device Manager](https://github.com/kubernetes/kubernetes/blob/v1.32.0/pkg/kubelet/cm/devicemanager/manager.go)、[NVIDIA Device Plugin](https://github.com/NVIDIA/k8s-device-plugin) | 节点数量与具体设备分别由谁决定？插件失联与卡 unhealthy 是同一回事吗？分配状态如何恢复？ |

每周附一张简短的设计记录：问题、关键机制、替代方案、当前取舍、失败时的行为、对自己的工具有何启发。第 2 月起至少对一处改动运行相关测试或行为实验，避免只凭解释判断理解程度。

Go 在上述源码和项目预算内随任务补齐：第 1～4 周掌握 interface、error、context、goroutine 的退出、map/slice 与并发保护；第 5～8 周练表驱动测试、依赖注入、超时和重试；第 9～12 周落实包边界、配置、结构化日志与竞态检测；第 13～16 周使用 benchmark/pprof 验证一个真实瓶颈。每次只补足完成当周任务需要的知识，并通过自己的代码解释机制。

APIserver 只沿一次请求理解认证/授权、准入、存储调用、resourceVersion 与冲突；etcd、CNI、CSI 以排障边界为主。四个月后再选择其中一个继续深入。

## 6. 结业项目：GPU 节点验收与作业编排工具

### 必做范围

1. 验收采集器：记录机器/GPU/NIC/软件栈、拓扑、诊断覆盖、计算结果和原始证据；生成 JSON 与 Markdown 报告。
2. GpuNodeReport：导入验收结果，记录硬件能力、健康结论、时间戳、原因和报告引用。先采用离线导入，长期驻留采集 Agent 是后续增强项。
3. GPUWorkload：描述镜像、命令、整卡数、型号/节点池等约束，由 client-go 控制器创建 batch Job、跟踪状态并解释失败。主线限定每节点 GPU 型号同质、整卡独占且未启用共享，型号约束通过节点标签表达。混卡节点统一使用 `nvidia.com/gpu` 时，标签只能约束节点，无法保证节点内分配到指定型号，留待后续设备模型设计。
4. 节点准入：在自有字段/标签范围内表达合格状态；新任务结合资源请求、亲和性和默认调度器选择节点。
5. 运维入口：CLI、配置、日志、基础指标、部署清单、版本记录、暂停与回退说明。

机器学习侧用一个固定的小型训练或推理 Job 验证“提交 → GPU 可用 → 正确结果/产物 → 状态与日志 → 失败定位”。模型业务只需理解输入、输出、显存需求和任务生命周期。完整的数据集管理、实验追踪、计费和门户留到第二阶段。

### 三层职责

| 模块 | 本计划中的职责 |
| --- | --- |
| client-go 控制器 | 观察状态、处理队列、协调报告和 Job、维护业务约束 |
| 默认调度器 / Scheduler Framework | 为 Pod 选择可行节点；进阶插件可补充健康过滤或排序 |
| kubelet / NVIDIA Device Plugin | 在传统 Device Plugin 路径中发现、发布和分配节点内设备，配置容器访问 |

不要通过给业务 Pod 直接设置 `nodeName` 来冒充调度插件。`nvidia.com/gpu: 1` 表达整卡数量，不直接承诺某个 UUID、指定显存或最佳 GPU–NIC 组合。[Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)、[Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

### 工程验收重点

- 幂等：同一事件反复到达，不重复创建任务或执行外部动作。
- 恢复：重启、API 短暂中断、更新冲突、任务删除后，状态能重新收敛。
- 缓存：初始同步后开始工作，不原地修改共享缓存对象；通过索引减少扫描。
- 可观测：能看到队列深度、重试、协调耗时、失败原因、API 请求及采样新鲜度。
- 权限和归属：操作范围明确，仅管理自己的字段；对象清理有 owner reference，确有外部清理需求再使用 finalizer。
- 健康策略：采样缺失/过期是未知；恢复需要复验；频繁变化要有去抖和冷却机制。
- 一致性边界：标签和缓存的传播存在延迟，需测量并说明准入窗口；设备最后一刻异常仍可能在 kubelet/插件阶段失败。
- 调度热路径保持轻量：如做插件，使用缓存中的状态，不同步跑验卡、SSH 或查询监控服务。

主线使用已有报告和实验事件验证状态转换。异常处置先输出建议、标记和限制新任务，自动驱逐现有业务、GPU reset、自动重启及重装属于另行设计的变更能力。

### 18 小时增强项

按优先级选择，不要求全部完成：

1. 针对 v1.32 构建独立 `schedulerName` 的实验调度器，保留默认资源/污点/亲和性约束，实现一个 Filter 或 Score 插件。对不健康/过期节点做过滤或对固定策略排序，验证无可行节点时的可解释性。
2. 两台机器可同时借到时，完成 RoCE 主机内存 RDMA、具备条件时 GPU 内存路径、NCCL 正确性与选路对照。
3. 选择一处真实问题提交小型修复或回归测试；不把上游 PR 合并作为结业前提。
4. 阅读 MIG/共享与 DRA 的能力边界。v1.32 的 DRA 为 beta 且默认关闭，主线采用 Device Plugin；不要用当前版本 DRA 教程直接套到 v1.32。[v1.32 DRA 文档](https://v1-32.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)

## 7. 四次 GPU 实机窗口

借卡前完成：确认型号与权限、准备版本矩阵、预拉镜像、预编译匹配架构的测试程序、准备 YAML、测试采集脚本。争取至少四次 3 小时窗口；没有拿到的实验明确保留待验状态。

| 窗口 | 目标 | 3 小时预算 |
| --- | --- | --- |
| A：底层链路 | GPU 档案、driver、Toolkit/runtime、容器正确计算 | 20 分钟盘点；50 分钟环境就绪；45 分钟 runtime；35 分钟计算验证；30 分钟证据/交接 |
| B：K8s 链路 | 资源发布、GPU Pod、一次可恢复配置故障 | 20 分钟复核；45 分钟插件注册；45 分钟 Pod；40 分钟故障定位；30 分钟证据/交接 |
| C：验收基线 | PCIe/NUMA、带宽/计算、指标与适用的诊断 | 25 分钟拓扑；60 分钟重复测量；35 分钟负载采样；30 分钟诊断；30 分钟证据/交接 |
| D：项目回归 | 候选版本实机部署、Job、健康导入和恢复验证 | 20 分钟复核；45 分钟部署/报告；45 分钟任务闭环；40 分钟异常/恢复；30 分钟证据/交接 |

窗口 A 若需要完整重装，不能保证 3 小时完成全部环节。单个兼容问题定位约 30 分钟仍未解决时，保存证据，完成可做的下层验收，后续项标记前置条件不足。不要用完窗口才发现没有保留版本和错误原文。

每次至少保存：机器档案、全部命令及退出码、内核日志时间段、容器/插件/Pod 日志、Events、测试输入输出、指标时间线和配置差异。原始证据分析可在归还机器后继续。

### 新机报告的必填项

- 身份与配置：GPU/NIC 完整型号、序列或 UUID、固件、PCIe/NUMA、系统和软件版本。
- 能力与兼容性：ECC/MIG/DCGM/RDMA 的适用条件；不适用项有依据。
- 逐层检查：宿主机 → 容器 → K8s 资源 → 实际 GPU 计算；不只看设备可见。
- 基线：同配置下至少重复 3 次适用测试，记录中位数、波动、温度/功耗/频率及错误计数前后差值。
- 结论：PASS / WARN / FAIL / UNSUPPORTED / NOT_TESTED / INCONCLUSIVE；逐项附证据、影响与复验条件。

性能阈值依据同型号、同软件和拓扑配置的健康基线。短时测试通过只证明测试覆盖范围内未发现异常。

GPU 运维特别容易误解的四件事：

1. `nvidia-smi` 顶部 CUDA Version 是驱动支持的最高 CUDA 版本；Toolkit 和容器 runtime 需另查并实测。[官方兼容说明](https://docs.nvidia.com/datacenter/tesla/drivers/latest/cuda-toolkit-driver-and-architecture-matrix.html)
2. ECC/DCGM 字段 N/A 不能当成零错误；诊断覆盖随型号和版本变化。[DCGM 功能矩阵](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html)
3. Xid 是排障起点，需要结合上下文；不能只凭一个编号判换卡。[Xid 分析](https://docs.nvidia.com/deploy/xid-errors/analyzing-xid-catalog.html)
4. RoCE 连通、GPU 内存 RDMA、NCCL 实际使用某条路径需要分别证明；具备 RDMA NIC 不足以证明 GDR 可用。[GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)、[NCCL Tests](https://github.com/NVIDIA/nccl-tests)

## 8. 月考与结业验收

考核时间使用当周项目/复盘预算；下列时限是学习目标，第一次超时用于发现缺口。

| 时点 | 考核任务 | 通过条件 |
| --- | --- | --- |
| 第 4 周 | 演示小控制器；解释 API → informer → queue → 协调 → API；定位一个状态不更新问题 | 有实际运行证据；能解释缓存、重复事件、重试、退出；60 分钟内形成证据链 |
| 第 8 周 | GPU 实机分层验收；抽 3 个层次故障；讲 GPU Pod 完整链路 | 至少一次真实 GPU 闭环；能区分 Pending 和已调度后失败；源码位置与日志相互对应 |
| 第 12 周 | 演示 v0.1 的恢复和工程能力 | 重复事件、409、重启、API 中断、Job 超时/删除、双副本切换等核心异常路径通过；含真实 API Server 验证 |
| 第 16 周 | 完整工具演示、陌生故障、部署交接 | 90 分钟内给出有证据的定位、处置和复验；另一人能按文档部署；未验能力明确列出 |

最终 8 个核心异常场景：

1. 同一工作负载被反复入队。
2. API 更新发生冲突。
3. 控制器处理中重启。
4. API Server 短暂不可达。
5. Job 失败、超时或被外部删除。
6. 健康证据过期或采样中断。
7. 节点从合格变为疑似异常，再经复验恢复。
8. 两副本发生领导者切换。

真实 API Server 用于验证 watch、鉴权、冲突和协调；fake client 用于业务逻辑测试。用模拟报告/Node/Pod 做规模试验时，先小后大，记录对象类型、数量、事件速率、环境配置、缓存就绪、内存、API QPS、队列时延和恢复时间。模拟对象测试结果只支持相应结论，真实 GPU 与真实大集群集成另记。

试点分为只读观察、指定测试节点写入、少量测试工作负载三步。每步明确范围、准入指标、暂停和回退条件。四个月可交付“生产候选版本 + 验证报告”；生产成熟度还需要目标版本、真实负载和持续运行的验证。

## 9. 第一周直接执行

逐次学习的文档章节、命令、实现步骤与验收详见 [第一周执行手册](/Users/wty/workspace/study/week01/第一周执行手册.md)。该手册仍按 14 小时主线执行，实际学习日期由你指定。

| 学习块 | 时间 | 具体动作 | 产出 |
| --- | ---: | --- | --- |
| 工作日 A | 2 小时 | 30 分钟记录基础环境和版本；60 分钟读 Go interface/error/context 并修改最小示例；30 分钟写疑问 | 环境清单、能超时退出的小程序 |
| 工作日 B | 2 小时 | 45 分钟追一次 Pod/Node 查询；45 分钟用 Linux namespace/cgroup 观察进程或容器；30 分钟画图 | 初版调用图、一份命令/现象对应记录 |
| 工作日 C | 2 小时 | 60 分钟读 GPU 软件栈和能力字段；60 分钟设计机器档案及窗口 A 准备包 | GPU 能力表、所需包/镜像/权限清单 |
| 周末 | 8 小时 | 源码/目标包测试 1 小时；GPU 准备 1 小时；只读 CLI 开发与验证 4 小时；互动复盘/补漏 2 小时 | 查询 Node/Pod 并输出 JSON 的 CLI、实验记录、待解决问题 |

CLI 第一版仅做：指定 kubeconfig/context、超时和取消、列出 Node 的 GPU capacity/allocatable、列出目标范围 Pod 的 GPU 请求、JSON 输出与清楚报错。没有 GPU 的节点按正常情况处理。先通过只读功能理解 API，不在第一周加入自动隔离或调度。

第一周自测问题：

- 对某个 Pod，API 对象存在、完成调度、容器运行分别由哪些证据证明？
- 进程在容器内看到的 PID 与宿主机 PID 为什么可能不同？
- capacity、allocatable 与已经申请的 GPU 数量分别是什么意思？
- context 超时后，哪些调用应该停止？
- 机器上 `nvidia-smi` 正常，为什么仍不能断定 GPU Pod 能正确计算？

## 10. 和助手持续对齐的方法

每次学习结束，把“本次目标、实际完成、代码/日志证据、自己的解释、卡住的问题”发到同一个聊天。助手先根据真实内容提 3～5 道题，每轮最多 2 道；等你回答后再纠错和追问，最后共同复盘。没有记录的学习不自动当作已完成。

评价分为“有证据已验证 / 需巩固 / 未验证”，分别衡量解释机制、采集证据、完成实现、迁移排障的能力。对不熟悉的问题，先给提示；你明确需要时再给完整答案。

每周复盘一次：核对实际投入、验证当周产出、找一个主要缺口、确定下周三个必做任务。工作事故可作为学习材料，但需整理成可复现且脱敏的证据，不用生产试错补实验。

进度、工作日安排和提醒偏好记录在 [学习进度.md](/Users/wty/workspace/study/学习进度.md)。你提前告知工作日学习日期后，再按实际安排对齐；周内未指定的日期不推断为已学习。定时提醒的最终状态以工具登记结果为准。

2026-10-26 按用户要求增加每日自行翻译任务，主文档固定为 [Kubernetes API Concepts v1.32](https://v1-32.docs.kubernetes.io/docs/reference/using-api/api-concepts/)。具体选段、方法与记录见 [每日英文翻译任务](/Users/wty/workspace/study/translate/k8s-api-concepts.md)。

先试七天，每天 15 分钟，从 1～2 句、约 10～40 词起步；当天只理解一句也可以。每周 105 分钟计入源码/Go/容器机制的 5 小时预算，其余约 3 小时 15 分钟用于该类其他学习，整体仍为 14 小时。同一文档的技术学习与英文训练不重复计时；原来休息日的英文小任务从本周同一预算调剂。若影响源码实验，先缩为每日 5～10 分钟，再按真实进度调整。

每次最多记录 3 个有语境的词或搭配；重点练主语与动作、条件、否定和指代，自己尝试后再核对。七次后用重读和少量新句检验理解，不按字数或翻译页数判通过。当前只推进一篇主文档，既有 cgroup 草稿保留，GPU 词汇在后续专项阅读中补。一篇文档不承诺覆盖整个计算机专业的大多数词汇。

## 11. 按需查阅的官方资料

- 第 1 月：[client-go v0.32.0](https://github.com/kubernetes/client-go/tree/v0.32.0)、[sample-controller v0.32.0](https://github.com/kubernetes/sample-controller/tree/v0.32.0)、[Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)。API 文档按 v1.32 实现核对版本差异。
- 第 2 月：[Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)、[Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)、[NVIDIA Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)。Framework 与设备接口对照固定 tag 阅读。
- GPU 部署：[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)。安装前查当前 Debian/内核/GPU 支持及组件版本。
- GPU 验收与排障：[GPU Debug Guidelines](https://docs.nvidia.com/deploy/gpu-debug-guidelines/)、[Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)、[nvbandwidth](https://github.com/NVIDIA/nvbandwidth)。工具和测试覆盖按实际硬件核对。
- GPU 条件进阶：[MIG 前提](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/getting-started-with-mig.html)、[GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)、[NCCL Tests](https://github.com/NVIDIA/nccl-tests)。

计划中的任务、课时与验收门槛是针对你当前情况的设计；链接用于核对机制、接口和支持边界。第一次周复盘后，依据实际完成速度调整第 2～4 周的任务量。
