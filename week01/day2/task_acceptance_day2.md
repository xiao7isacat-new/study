# 任务验收

## k8s 集群外访问list 代码追踪

### 配置创建

1. 调用 BuildConfigFromFlags 函数 。传入kubeconfig的配置，
2. 调用NewNonInteractiveDeferredLoadingClientConfig函数。返回一个&DeferredLoadingClientConfig 的 interface。这个DeferredLoadingClientConfig 结构体实现了ClientConfig 这个interface。
3. 调用DeferredLoadingClientConfig 的 ClientConfig 方法生成最终的*rest.Config。

### 创建config 的集合

调用kubernetes.NewForConfig 函数，生成一个http 的客户端集合。

### 启动一个永远为true 的for循环。在循环中做如下的事情

1. 调用ClientConfig 的CoreV1().Pods("").List 函数来查看所有的pod。
   这个函数传入一个context。和一个listoptions 的结构体。
2. 开始list（这里的list 和 服务端的watch-list不是一个。WatchListClient 默认关闭。启用且条件适用时才尝试 watch-list；否则进入普通 list）,最终返回一个结构体。结构体有一个item 列表返回的是pod 的各种字段新。
3. CoreV1() 返回的就是kubernetes.Clientset 这个 struct 的coreV1资源。

### do 和info 的边界

```
  198     err := l.client.client.Get().
  199         UseProtobufAsDefaultIfPreferred(l.client.prefersProtobuf).
  200         NamespaceIfScoped(l.client.namespace, l.client.namespace != "").
  201         Resource(l.client.resource).
  202         VersionedParams(&opts, l.client.parameterCodec).
  203         Timeout(timeout).
  204         Do(ctx).
  205         Into(list)
```

do 是用来发送请求的，返回一个rest.Result

into事用来解析返回的内容的，对rest.Result 本身的数据做处理,只会返回error。 into 会调用自己的decoder的decode方法，来解析rest.Result 的body，将内容映射到into 函数传入的 名为list 的runtime.Object 这个interface 中

## namespace 与 cgroup 观察

### uts试验

```
root@k8s01:~# hostname
k8s01
root@k8s01:~# readlink /proc/$$/ns/uts
uts:[4026531838]
root@k8s01:~# unshare --uts -f sh -c 'hostname k8s-week1; hostname; readlink /proc/self/ns/uts'
k8s-week1
uts:[4026532732]
root@k8s01:~# hostname
k8s01
root@k8s01:~# readlink /proc/$$/ns/uts
uts:[4026531838]
```

使用unshare 命令。指定一个新的uts之后隔离了主机名称。启动了一个新的namespace system resource。所以hostname 前后的主机名都是本机的名称。之后unshare 中指定的主机名输出的是k8s-week1

### cgroup

cgroup相关的文档太长了，而我的英文能力很有限。所以我打算通过自己翻译英文文档来提升自己的英文水平。

刚好cgroup 文档我认为是一个不错的文档，可以用来联手。但我的翻译时间就会变的很慢了。我预计需要一周的时间来翻译。

```
root@k8s01:~# stat -fc %T /sys/fs/cgroup
cgroup2fs
root@k8s01:~# cat /proc/$$/cgroup
0::/user.slice/user-0.slice/session-542.scope
root@k8s01:~# echo $STUDY_CGROUP_PATH
/user.slice/user-0.slice/session-542.scope
root@k8s01:~# for STUDY_CGROUP_FILE in cpu.max memory.max memory.current; do
    STUDY_CGROUP_FULL="/sys/fs/cgroup${STUDY_CGROUP_PATH}/${STUDY_CGROUP_FILE}"
    if [ -f "$STUDY_CGROUP_FULL" ]; then
        printf '%s: ' "$STUDY_CGROUP_FILE"
        cat "$STUDY_CGROUP_FULL"
    fi
done
memory.max: max
memory.current: 2715648
```

### pod

![image](file:////Users/wty/workspace/study/.images/week01/day2/task_acceptance_day2/image.png)

## 复盘问题：

1. `BuildConfigFromFlags`、`kubernetes.NewForConfig` 分别返回什么类型？哪一个返回值可以调用 CoreV1，为什么它不是控制器？

   `BuildConfigFromFlags` 返回的是restclient.Config。`kubernetes.NewForConfig` 返回的是kubernetes.Clientset。
   kubernetes.Clientset 可以调用CoreV1。他名称很明显的有一个标识说是client 。而不是server的控制器。
2. 一个 Pod 已有 nodeName 且 PodScheduled=True，但 phase 仍是 Pending，是否矛盾？你会从哪些实际字段或事件继续定位？
   我认为是不矛盾的。有 nodeName 且 PodScheduled=True。已经有nodeName 说明schedule 经过打分之后认为node上的资源是符合预期的。我会根据 event 事件来查看这个pod 否发生了哪些事件，是否因为磁盘，cpu内存等问题导致他又变得不可调度。
3. Pod 已绑定 worker-1，仍 Pending，原因是 `ErrImagePull`。修改 scheduler 打分能直接解决吗？你会先查哪两类信息？
   看情况而定。因为调度器的打分只能解决调度的问题。如果修改打分调度的其他节点。而其他节点可以拉镜像，那就可以解决。如果还是在本节点上。无法拉去镜像是 container 层面的事情，和k8s 的 schedule 没有关系。
