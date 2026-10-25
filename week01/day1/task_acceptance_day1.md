# 任务验收

## 基础环境信息记录

```shell
my-ubuntu-01 是我的开发机。
k8s01 是k8s集群的第一个master

root@my-ubuntu-01:/opt/workspace/study/day1# go version
go version go1.27.0 linux/amd64

root@k8s01:~# kubectl version --client
Client Version: v1.32.13
Kustomize Version: v5.5.0

root@k8s01:~# cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.1 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy

root@k8s01:~# uname -m
x86_64

root@k8s01:~# crictl --version
crictl version v1.32.0

root@k8s01:~# mount | grep cgroup
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)

root@k8s01:~# kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@offline-ha   offline-ha   kubernetes-admin


root@k8s01:~# kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@offline-ha   offline-ha   kubernetes-admin

root@k8s01:~# kubectl --context "$STUDY_KUBE_CONTEXT" version
Client Version: v1.32.13
Kustomize Version: v5.5.0
Server Version: v1.32.13

root@k8s01:~# kubectl --context "$STUDY_KUBE_CONTEXT" auth can-i list nodes
Warning: resource 'nodes' is not namespace scoped
yes


root@k8s01:~# kubectl --context "$STUDY_KUBE_CONTEXT" auth can-i list pods -n "$STUDY_NAMESPACE"
yes

root@k8s01:~# kubectl --context "$STUDY_KUBE_CONTEXT" get nodes -o wide
NAME      STATUS   ROLES           AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master1   Ready    control-plane   77m   v1.32.13   172.16.0.6    <none>        Ubuntu 22.04.1 LTS   5.15.0-91-generic   containerd://1.7.36
master2   Ready    control-plane   76m   v1.32.13   172.16.0.7    <none>        Ubuntu 22.04.1 LTS   5.15.0-91-generic   containerd://1.7.36
master3   Ready    control-plane   76m   v1.32.13   172.16.0.8    <none>        Ubuntu 22.04.1 LTS   5.15.0-91-generic   containerd://1.7.36

root@k8s01:~# kubectl --context "$STUDY_KUBE_CONTEXT" get pods -n "$STUDY_NAMESPACE" -o wide
No resources found in default namespace.

root@k8s01:~# containerd --version
containerd github.com/containerd/containerd v1.7.36 2892c2042ee7fbd3be0e5bdc675e07b5acedb0bf

root@k8s01:~# stat -fc %T /sys/fs/cgroup
cgroup2fs

```

## interface 和error 的基础使用代码

```go
package main

import (
        "context"
        "fmt"
        "time"
)

type animals interface {
        eat()
        sleep()
}

type dog struct {
}

func (this *dog) eat() {
        fmt.Println("dog eat")
}

func (this *dog) sleep() {
        fmt.Println("dog sleep")
}

type cat struct {
}

func (this *cat) eat() {
        fmt.Println("cat eat")
}

func (this *cat) sleep() {
        fmt.Println("cat sleep")
}

func getAnimals(animal string) (animals, error) {
        if animal == "dog" {
                return &dog{}, nil
        }
        if animal == "cat" {
                return &cat{}, nil
        }
        return nil, fmt.Errorf("not exist")
}

func work(ctx context.Context, cost time.Duration) error {
        timer := time.NewTimer(cost)
        defer timer.Stop()
        select {
        case <-ctx.Done():
                return fmt.Errorf("work stopped: %w", ctx.Err())
        case <-timer.C:
                return nil
        }
}

func main() {
        now := time.Now()
        ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
        defer cancel()
        err := work(ctx, 300*time.Millisecond)
        fmt.Printf("result=%v elapsed=%v\n", err, time.Since(now))

        animal, err := getAnimals("duck")
        if err != nil {
                fmt.Println(err)
        }
        animal, err = getAnimals("dog")
        if err != nil {
                return
        }
        animal.sleep()
        animal.eat()
        animal, err = getAnimals("cat")
        if err != nil {
                return
        }
        animal.sleep()
        animal.eat()

}
```

## context实验结论

- [X] 保持 100ms 超时、300ms 工作量，记录结果与大致耗时。

```
result=work stopped: context deadline exceeded elapsed=100.240779ms
not exist
dog sleep
dog eat
cat sleep
cat eat
```

大约100ms超时。因为work函数中的ctx 100ms到时间了。所以work函数返回work stopped

- [X] 将超时改为 500ms，其他不变，记录差异。

```
result=<nil> elapsed=300.425936ms
not exist
dog sleep
dog eat
cat sleep
cat eat
```

大约300ms超时。因为work函数中的timer.c 模拟任务正常执行完成了。所以返回nil。程序接着往后跑。

- [X] 在调用 `work` 前主动 `cancel()`，记录错误。

```
result=work stopped: context canceled elapsed=26.712µs
not exist
dog sleep
dog eat
cat sleep
cat eat
```

26us。因为执行cancel() 之后，对应done 的channel会被关闭，work函数中的ctx.Done()会收到信号。所以work函数 会直接输出work stopped。

## 问答

* 如果删除 `work` 中整个 `case <-ctx.Done()` 分支，只保留 timer 分支，仍设置 **100ms 截止、300ms 工作量**，它会怎样返回？为什么？

```
他只会等work 函数的cost 参数时间到了之后返回也就是300ms 返回工作量。
因为select 的时候只有timer.c 会到300ms 之后输出并且返回nil。
```

* 如果把 `return &dog{}, nil` 改成 `return dog{}, nil`，其他代码不变，能否编译？请结合方法声明说明理由。

```
不能编译。
因为函数返回的是一个 指针类型。而&dog{} 完成了 对应指针的eat 和 sleep 的方法 dog{} 并没有完成对应的方法。
```
