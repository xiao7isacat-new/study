# Kubernetes v1.32 三主离线部署使用手册

更新日期：2026-10-22。适用原源码包：`k8s-v1.32-ubuntu22.04-3master-havip-offline-source.tar.gz`。

本文使用**联网准备机上已经构建成功的完整离线包，部署三台新的Ubuntu 22.04机器**。准备机仅提供物料，新master1负责执行Ansible并将物料分发给另外两台。全部操作不依赖原来的m1或任何以前集群的文件。

本文只整理使用步骤和原工程内的必要手工修正，不提供新源码包，不重新下载镜像，不新增pip wheel。下文所有m1/master1均指本次的新master1。

## 1. 部署目标与阅读入口

### 1.1 本次拓扑

| 机器/地址 | 用途 |
|---|---|
| 现有联网准备机 | 保存原包构建成功的完整工程和artifacts，校验、打包、传输 |
| master1：172.16.0.6 | 控制平面、本地etcd、业务Pod、Ansible执行端 |
| master2：172.16.0.7 | 控制平面、本地etcd、业务Pod |
| master3：172.16.0.8 | 控制平面、本地etcd、业务Pod |
| veStack HaVIP：172.16.0.4 | Kubernetes API入口，TCP8443 |

以上主IP来自你最近提供的日志。若用于另一组三台机器，安装前在inventory.ini统一改成真实固定主IP；HaVIP在config/cluster.yml配置。准备机不填入inventory，也不加入集群。

API路径：`172.16.0.4:8443 → 当前VIP节点的HAProxy → 三台节点各自:6443`。三台都允许业务Pod调度，不另加worker。Keepalived使用单播，master1初始MASTER，优先级100；另外两台BACKUP，优先级50/25；允许抢占。

### 1.2 选择入口

**继续当前机器的验收：** 你的最近日志已经进入etcd验收，失败是JSON传参格式。核对**第8.3节**的两行修正，再执行**第12节**的verify.yml即可。不要重新解压覆盖工程、重建.venv、安装全部依赖或重新init/join，不需要reset。

**将来从三台空白系统重新部署：** 按下表从第2节顺序执行。每节达到继续条件后再进入下一节，不跳过依赖模拟或配置检查。

| 顺序/章节 | 执行位置 | 操作与继续条件 |
|---|---|---|
| 2 | 准备机 | 现成完整包校验通过；仅有成功工程目录时使用原package.sh打包 |
| 3 | 三台新机器、新master1 | OS/HaVIP检查，完整包传入m1、解压并逐项校验 |
| 4 | 新master1 | 离线建立或检查Ansible 2.17.14 |
| 5 | 新master1 | 配置三台IP、共同网卡、HaVIP和网段，SSH/Python连通 |
| 6 | 三台/云控制台 | 核对主机防火墙、云安全组、VRRP和HaVIP漂移能力 |
| 7 | 新master1→三台 | 仅必要OS包，先模拟零升级/零删除再安装 |
| 8 | 新master1 | 原包版本正则、swap查询、etcd JSON传参修正 |
| 9 | 新master1 | Keepalived最终模板、脚本和部署任务，备份/回滚路径配套 |
| 10 | 新master1→三台 | 独立预检查，三台failed=0、unreachable=0 |
| 11 | 新master1→三台 | 原site.yml完成三台部署和正常验收 |
| 12 | 新master1→三台 | 独立复验，确认API、etcd、跨节点网络、服务自启 |
| 13 | 新master1→三台 | 正常验收后做VIP故障切换及逐台重启 |
| 14–15 | 按需 | 故障定位、明确拆除整个集群时的基础回滚 |

**原包安装入口为site.yml，验收入口为verify.yml。** 不使用不存在的分阶段标签或keepalived.yml；不执行全量bootstrap_offline.sh/bootstrap.yml，也不全量安装artifacts/debs。目标节点的所有安装均使用本地物料。

## 2. 联网准备机：校验现成物料并传给新master1

本节所有命令都在**已有原包构建成果的联网准备机**执行。使用现成成果，不运行prepare.sh。

需要区分两个文件：

- `k8s-v1.32-ubuntu22.04-3master-havip-offline-source.tar.gz`是原始源码包，本身没有完整镜像、Deb和wheels，不能单独用来离线安装。
- **已构建成功的完整离线包**包含原工程及artifacts，才是要传给新master1的文件。原package.sh默认完整包名是`k8s-offline-ubuntu22-ha-amd64.tar.gz`；以下以这个名字为例，实际文件名不同则统一替换。

### 2.1 准备机上已经有完整tar包：直接校验

默认文件位于准备机/opt；实际位置不同则先进入对应目录：

```bash
cd /opt
sha256sum -c k8s-offline-ubuntu22-ha-amd64.tar.gz.sha256
tar -tzf k8s-offline-ubuntu22-ha-amd64.tar.gz | grep -E '/artifacts/(lock.yml|SHA256SUMS)$'
```

继续条件：包的SHA256通过，归档里同时有artifacts/lock.yml和artifacts/SHA256SUMS。若只是原始源码包，不会具备这两个完整物料文件。完整性还会在新master1解压后逐项验证。package.sh若提示Output exists，直接校验现有包，不覆盖。

**已有完整包直接进入2.3，不再打包或重新构建。**

### 2.2 准备机只有构建成功的工程目录：仅打包现有成果

仅当2.1的完整tar包尚未生成时，在准备机中已有完整artifacts的工程根目录执行；下面路径指**准备机上的目录**：

```bash
cd /opt/k8s-offline-ubuntu2204-ha
python3 scripts/manifest.py verify artifacts
sudo bash scripts/package.sh /opt/k8s-offline-ubuntu22-ha-amd64.tar.gz
cd /opt
sha256sum -c k8s-offline-ubuntu22-ha-amd64.tar.gz.sha256
```

package.sh只是把**这台准备机现有的原工程和成功物料**封装为可搬运文件，不下载、不换源码。若提示输出文件已存在，回到2.1检查已有包，不覆盖。准备机工程目录不同时按实际路径进入；若归档顶层名称不同，第3节相应调整新master1上的工程路径。

完整包必须保留整个artifacts目录，以及配套的config/versions.env和config/downloads.env，不挑选文件或混入其他版本。新的inventory和HaVIP配置在新master1上按第5节填写。

### 2.3 从准备机传到新master1

先通过云控制台核验新master1的SSH指纹。下面使用本次新master1主IP172.16.0.6；地址变化时按实际修改，不能填HaVIP。例子假设root可SSH登录，其他账号请改成实际账号并使用其可写目录，随后通过sudo将包放到新master1的/opt。

```bash
# 仍在联网准备机上执行；先替换目标IP。
scp /opt/k8s-offline-ubuntu22-ha-amd64.tar.gz \
    /opt/k8s-offline-ubuntu22-ha-amd64.tar.gz.sha256 \
    root@172.16.0.6:/opt/
```

准备机无法直连目标时，可通过可用的文件传输渠道或移动介质搬运这两个文件。**只需把完整包搬到新master1；后续由新master1的Ansible将物料分发给新master2、master3。** 准备机保留其成功成果即可，不加入集群，不把准备机的.venv复制给目标。

## 3. 三台新机器检查，新master1解压完整包

每台至少 2 vCPU、约 2 GiB 内存、安装前 /var/lib 空闲 20 GiB；三主兼业务建议至少 4 vCPU/8 GiB、可靠 SSD，并按业务容量增加。

在三台各自检查，不安装任何东西：

```bash
cat /etc/os-release
uname -m
uname -r
python3 --version
systemctl --version | head -1
cat /sys/fs/cgroup/cgroup.controllers
ip -o -4 address show scope global
ip -4 route
timedatectl status
df -h /var/lib
dpkg --audit
```

要求三台均为Ubuntu 22.04、x86_64、Python 3.10、systemd、内核>=5.15，统一cgroup v2包含cpu/memory/pids。三台固定主IP和主机名各自唯一，时间同步；离线时使用可达的内网时间源。不要预装Docker/containerd/Kubernetes。

在veStack控制台确认**172.16.0.4关联当前这三台新实例/网卡**，并满足单播VRRP、GARP和地址漂移条件。若目前关联其他实例，需在控制台调整为当前三台；IP数值相同不代表关联自动转移。完成后才能将第5节ha_cloud_havip_associated设为true。

在新 m1 的 /opt 接收第 2 节完整搬运包及校验文件，目标目录不存在时解压：

```bash
cd /opt
sha256sum -c k8s-offline-ubuntu22-ha-amd64.tar.gz.sha256
test ! -e /opt/k8s-offline-ubuntu2204-ha && tar -xzf k8s-offline-ubuntu22-ha-amd64.tar.gz -C /opt
cd /opt/k8s-offline-ubuntu2204-ha
python3 scripts/manifest.py verify artifacts
```

若包顶层目录不是k8s-offline-ubuntu2204-ha，请先查看tar -tzf的目录结构并统一路径；不直接覆盖已有目录。**从此处开始，后续没有特别说明的命令都在新master1的工程根目录执行，操作用户为root，不再回到准备机。** 三台初始均为空白目标，所有证书、集群状态和Ansible环境由本次步骤生成。

`Artifact checksums, lock, version configuration, and required files verified.`表示**校验成功，不是报错**。缺文件/hash不符时从准备机重新搬运同一完整包，不修改lock/SHA256绕过。

| 路径 | 保留内容 |
|---|---|
| artifacts/bin/ | kubeadm、kubelet、kubectl |
| artifacts/downloads/ | containerd.tar.gz、runc.amd64、cni-plugins.tgz、crictl.tar.gz |
| artifacts/images/ | 全部镜像 tar 和真实引用/版本/digest 清单 |
| artifacts/manifests/ | Flannel 上游清单、真实 kubeadm 镜像清单 |
| artifacts/debs/ | 候选 OS 包，不能全部安装 |
| artifacts/wheels/ | Ansible 依赖 wheel、requirements.lock |
| artifacts/lock.yml、SHA256SUMS | 本次版本及完整性依据 |

| 组件 | 原包默认版本；最终以实际lock为准 |
|---|---|
| kubeadm / kubelet / kubectl | v1.32.13 |
| containerd / runc | 1.7.36 / 1.3.4 |
| crictl / reference CNI plugins | v1.32.0 / 1.7.1 |
| Flannel / Flannel CNI plugin | v0.28.4 / v1.9.1-flannel1 |
| pause / etcd / CoreDNS | 3.10 / 3.5.24-0 / v1.11.3，配合实际kubeadm清单 |
| Ansible-core | 2.17.14，仅在新master1的.venv中运行 |

镜像以原始引用导入containerd的k8s.io namespace。registry.k8s.io标签/后端会变化，使用现有包的真实清单与hash；不把传输加速器名称改成运行镜像名。

[Kubernetes1.32 已于2026-02-28 EOL](https://kubernetes.io/releases/1.32/)，[containerd1.7 已于2026-09-30 EOL](https://github.com/containerd/containerd/blob/main/RELEASES.md)。本文沿用你指定且已准备的版本，不在重建过程中换版本。

## 4. 新master1离线建立Ansible

### 4.1 先检查Ansible；已有可用环境就跳过安装

当前工程已有.venv时，先执行：

```bash
env -u PYTHONPATH .venv/bin/ansible-playbook --version
```

若core版本与lock一致，且解释器/Ansible模块路径都位于当前工程.venv，直接进入第5节。当前已经部署到验收阶段的机器不重建.venv。

首次安装时再检查系统Python工具：

```bash
python3 --version
python3 -m pip --version
python3 -c 'import venv; print("venv 模块可用")'
```

三条成功则进入4.3。`ensurepip`缺失不影响下面的方法；但pip与venv模块本身必须可用。全新机器按实际输出判断，不预设已经安装。

### 4.2 仅当 m1 没有系统 pip：用已有 Deb 补齐，不下载新 wheel

只在 m1 操作。先读取现有 pip 包声明：

```bash
dpkg-deb -f artifacts/debs/python3-pip_*.deb Package Version Depends
```

下面六类包是 Ubuntu jammy 系统 pip 常见的依赖候选；以实际 Depends 和 APT 模拟为准。先挑出到独立目录，脚本要求每类恰好一个候选：

```bash
bash <<'BASH'
set -euo pipefail
test ! -e work/m1-python-debs
mkdir -p work/m1-python-debs
for pkg in python3-pip python3-setuptools python3-wheel python3-pkg-resources python3-distutils python3-lib2to3; do
  files=("$PWD"/artifacts/debs/"${pkg}"_*.deb)
  test "${#files[@]}" -eq 1 && test -f "${files[0]}" || { echo "检查 $pkg 是否需要且有唯一候选，未继续安装" >&2; exit 1; }
  dpkg-deb -f "${files[0]}" Package Version Depends
  cp -a "${files[0]}" work/m1-python-debs/
done
BASH

mkdir -p /tmp/k8s-minimal-apt/empty /tmp/k8s-minimal-apt/lists/partial
env LC_ALL=C apt-get \
  -o Dir::Etc::sourcelist=/dev/null \
  -o Dir::Etc::sourceparts=/tmp/k8s-minimal-apt/empty \
  -o Dir::State::lists=/tmp/k8s-minimal-apt/lists \
  --simulate --no-download --no-remove --no-install-recommends \
  install "$PWD"/work/m1-python-debs/*.deb
```

**仅当0 upgraded、0 to remove、没有降级/冲突，且新增项只是已审核的 Python 纯包依赖时才安装。** 已安装的纯包候选若版本不同会导致升级/降级，可从这个工作目录排除该候选，再模拟系统当前版本是否足够；不要删除 artifacts 中的文件。不能因为缺 pip 就升级 libc/systemd/Python/OpenSSH。

模拟通过后：

```bash
dpkg -i "$PWD"/work/m1-python-debs/*.deb
dpkg --audit
python3 -m pip --version
python3 -c 'import venv; print("venv 模块可用")'
```

若候选缺失、还需其他包或 import venv 仍失败，停在此步，为该系统基线选择兼容离线依赖；不能保证六个候选覆盖所有精简镜像，也不能强装高版本 python3.10-venv。目标不在线 apt/pip install，不用 --force-depends 或在线 apt -f install。

### 4.3 创建 .venv，临时借用系统 pip 安装本地 Ansible wheels

```bash
test ! -e .venv && python3 -m venv --without-pip .venv
.venv/bin/python -c 'import sys; assert sys.prefix != sys.base_prefix; print(sys.prefix)'

PYTHONPATH=/usr/lib/python3/dist-packages PIP_CONFIG_FILE=/dev/null .venv/bin/python -m pip \
  --isolated --require-virtualenv --disable-pip-version-check \
  install --ignore-installed --no-index --no-cache-dir --only-binary=:all: \
  --find-links="$PWD/artifacts/wheels" --require-hashes \
  -r "$PWD/artifacts/wheels/requirements.lock"

env -u PYTHONPATH .venv/bin/ansible-playbook --version
```

成功标准：core2.17.14 与 lock 一致，Ansible 模块和解释器路径都在当前项目 .venv。PYTHONPATH 只用于这一条安装命令，不永久 export。不需要 ensurepip，也不要求向 .venv 单独安装 pip；以后直接 `.venv/bin/python -m pip` 可能仍提示 No module named pip，以 Ansible 可运行作为本节完成标准。

m2/m3 只需基础 Python/SSH，不需要安装 Ansible、pip 或 venv。

## 5. 配置真实 IP、SSH 和集群网络

### 5.1 inventory.ini 一次写全三台

```bash
cp -an inventory.example.ini inventory.ini
vi inventory.ini
```

本次三台日志对应如下配置，未来地址变化时安装前统一修改；各项使用节点主IP，不能填HaVIP：

```ini
[control_plane]
master1 ansible_host=172.16.0.6 kube_node_ip=172.16.0.6 ansible_connection=local
master2 ansible_host=172.16.0.7 kube_node_ip=172.16.0.7
master3 ansible_host=172.16.0.8 kube_node_ip=172.16.0.8

[all:vars]
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
```

只有在m1执行的master1使用local。三台名称/顺序从开始就固定，不通过先写一台、再添加另外两台来部署，不用--limit单台运行site。

在新master1首次连接另外两台时，先通过控制台核验它们当前的SSH主机指纹。若known_hosts存在冲突，核验后只更新对应记录，不关闭host_key_checking或清空全部记录。下面账号/IP按实际更改；root登录未开放时使用ubuntu等有sudo权限的账号，inventory同步修改：

```bash
test -s /root/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519
ssh-copy-id -i /root/.ssh/id_ed25519.pub root@172.16.0.7
ssh-copy-id -i /root/.ssh/id_ed25519.pub root@172.16.0.8
```

三台都检查：

```bash
env -u PYTHONPATH .venv/bin/ansible-inventory -i inventory.ini --graph
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -m ansible.builtin.ping
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'ip -o -4 address show scope global'
```

继续条件：三台ping:pong，主IP和接口正确。这里验证SSH/Python，不是ICMP。

### 5.2 修改 config/cluster.yml

保留config/cluster.yml中的其他键，修改已有键；新增Keepalived参数仅加一次。**ha_auth_pass必须加英文引号，作为字符串；同名键只能有一项。**

```yaml
ha_vip: 172.16.0.4
ha_vip_prefix: 32
ha_api_port: 8443
ha_router_id: 62
ha_cloud_havip_associated: true
ha_interface: eth0
flannel_iface: eth0
ha_priorities: [100, 50, 25]
ha_preempt_delay: 10
ha_auth_pass: '1111'
pod_cidr: 10.244.0.0/16
service_cidr: 10.96.0.0/12
cluster_dns: 10.96.0.10
allow_control_plane_workloads: true
coredns_upstream_dns: []
firewall_rules_acknowledged: false
```

eth0只是示例：三台主IP若都在ens3上，则两项都填ens3。**原包的Flannel必须填三台共同的真实内网接口名，不能留空或填m1 IP。** 留空会把m1 IP写进全局DaemonSet，其他两台无法按该地址选卡。接口名不一致先处理选卡方案，不直接初始化或随意重命名接口。

网段不能与VPC、主机、VPN、VIP及已有业务路由重叠。Service网段改动时DNS对应网络地址+10。CoreDNS默认仅解析集群名称，若要解析其他内网域名，在安装前填写可达内网DNS的IPv4列表。

VRID62需避开同网段其他VRRP组；三台auth_pass一致、用引号括起、1–8位字母数字，首次部署前可更改。PASS不是加密通道。

在新master1检查配置类型，不显示密码：

```bash
.venv/bin/python - <<'PYCODE'
from pathlib import Path
import re
import yaml
cfg = yaml.safe_load(Path('config/cluster.yml').read_text())
value = cfg.get('ha_auth_pass')
assert isinstance(value, str), "ha_auth_pass必须是加英文引号的字符串"
assert re.fullmatch(r'[A-Za-z0-9]{1,8}', value), "ha_auth_pass须为1–8位字母数字"
assert cfg.get('ha_priorities') == [100, 50, 25], "本手册优先级为100/50/25"
print('Keepalived参数类型与格式检查通过')
PYCODE
```

不要修改config/versions.env、config/downloads.env、artifacts/lock.yml或镜像名。集群安装后IP/VIP/网段等受到指纹保护，不通过删指纹来更换配置。

## 6. 网络、防火墙与云安全组

```bash
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'ufw status verbose'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'iptables-save'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'ip6tables-save'
```

系统没有ufw时跳过该工具，仍检查其他规则；有nft/firewalld时也核对实际配置。三台新机器分别检查，不预设UFW是否启用。预检查的active:ufw可能只是服务状态，需结合真实规则判断。

| 来源/目标 | 放行内容 | 用途 |
|---|---|---|
| m1→三台主IP | SSH TCP实际端口，默认22 | Ansible |
| 三台主IP相互 | TCP6443、8443、2379、2380、10250 | API/HAProxy/etcd/kubelet |
| 三台及受信管理端→HaVIP | TCP8443 | 稳定API入口 |
| 三台主IP相互 | **IP协议112**，不是TCP/UDP端口112 | Keepalived单播VRRP |
| 三台主IP相互 | UDP8472 | Flannel VXLAN |
| Pod/Service网络经主机转发链 | 必需双向转发、集群DNS UDP/TCP53等 | 跨节点Pod、ClusterIP、DNS |
| 节点→内网DNS/时间源 | 按实际DNS/NTP服务 | 名称与时间 |

云安全组不会因UFW inactive自动放行。veStack还需允许所关联HaVIP漂移、GARP；仅测试TCP不能证明VRRP/漂移可用。不要将etcd暴露公网。10257/10259默认本机使用；NodePort30000–32767按业务需要另配。此HaVIP只承载Kubernetes API，不代表业务Ingress已配置。

规则审核并准备完成后，将config/cluster.yml现有键改为：

```yaml
firewall_rules_acknowledged: true
```

这个值只是确认，不配置或关闭防火墙。不执行ufw reset、iptables -F或盲目停防火墙。

## 7. 三台最小OS依赖：先模拟，再安装

从本次搬入的成功物料中选择六个运行依赖候选，在三台新机器上分别模拟：haproxy、keepalived、conntrack、socat、libsnmp40、libsnmp-base。是否可安装以各台当前依赖求解为准。

先读实际lock中的HA软件版本：

```bash
python3 -c 'import json; print(json.load(open("artifacts/lock.yml"))["offline_lock"]["load_balancer_versions"])'
```

当前已知物料版本为HAProxy2.4.30-0ubuntu0.22.04.2、Keepalived1:2.2.4-0.2build1，必须核对准备机本次完整包中的实际lock。新m1只挑六个候选：

```bash
bash <<'BASH'
set -euo pipefail
test ! -e work/minimal-os-debs
mkdir -p work/minimal-os-debs
for pkg in libsnmp-base libsnmp40 haproxy keepalived conntrack socat; do
  files=("$PWD"/artifacts/debs/"${pkg}"_*.deb)
  test "${#files[@]}" -eq 1 && test -f "${files[0]}" || { echo "检查 $pkg 的唯一候选文件" >&2; exit 1; }
  dpkg-deb -f "${files[0]}" Package Version Depends
  cp -a "${files[0]}" work/minimal-os-debs/
done
BASH

env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.file -a 'path=/var/tmp/k8s-minimal-debs state=directory mode=0755'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.copy -a 'src=work/minimal-os-debs/ dest=/var/tmp/k8s-minimal-debs/ mode=0644'

env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.shell \
  -a 'mkdir -p /tmp/k8s-minimal-apt/empty /tmp/k8s-minimal-apt/lists/partial && LC_ALL=C apt-get -o Dir::Etc::sourcelist=/dev/null -o Dir::Etc::sourceparts=/tmp/k8s-minimal-apt/empty -o Dir::State::lists=/tmp/k8s-minimal-apt/lists --simulate --no-download --no-remove --no-install-recommends install /var/tmp/k8s-minimal-debs/*.deb'
```

逐台确认：**0 upgraded、0 to remove、无DOWNGRADED、无E:或Depends错误，只新增上述缺失包。** 目录里不得有其他Deb。已经满足的包允许0 newly installed。

三台模拟都通过后，使用普通dpkg安装同一组文件；这样也避开本地APT实装可能出现的Pathname to install is not absolute问题：

```bash
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.shell -a 'dpkg -i /var/tmp/k8s-minimal-debs/*.deb'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'dpkg --audit'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'systemctl stop haproxy keepalived'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command \
  -a 'dpkg-query -W -f="${Package}\t${db:Status-Status}\t${Version}\n" haproxy keepalived conntrack socat libsnmp40 libsnmp-base'
```

成功标准：三台六包installed、HA软件版本与lock精确相等、audit无错误；默认HA服务先停，等待工程写入实际配置。

检查其余工具：

```bash
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.shell \
  -a 'for t in ip ss iptables ip6tables iptables-save ip6tables-save conntrack socat ethtool mount umount swapon swapoff modprobe sysctl systemctl openssl tar gzip python3 findmnt sha256sum; do command -v "$t" >/dev/null || echo "MISSING:$t"; done'
```

缺ethtool/iptables等时，只为缺项从现有物料选择兼容Deb，另行模拟零升级/零删除后补齐。不保证六包覆盖所有精简镜像。出现基础库/Python/SSH升级要求先停，不全量安装artifacts/debs。

**本流程不执行原README中的scripts/bootstrap_offline.sh或playbooks/bootstrap.yml。** 全量候选依赖可能要求升级基础包，与本流程不符。site.yml只安装运行二进制与配置，不安装OS Deb。

## 8. 修正原包预检查与etcd验收传参

只在新m1修改本次从准备机完整包解压出的工程文件，不能只改本手册，也不需要以前部署时修改过的副本。按下面内容核对并修正；已一致的行保持即可。

```bash
mkdir -p work/manual-backup
cp -an playbooks/tasks/preflight.yml work/manual-backup/preflight.yml
vi playbooks/tasks/preflight.yml
```

### 8.1 版本正则：核对三行最终写法

在Require the declared three-control-plane target and exact lock format任务的that列表中，保持原缩进，改成：

```yaml
      - offline_lock.kubernetes_version is match('^v1[.]32[.][0-9]+$')
      - offline_lock.containerd_version is match('^1[.]7[.][0-9]+$')
      - offline_lock.crictl_version is match('^v1[.]32[.][0-9]+$')
```

这修正旧包过度转义导致合法版本被误拒绝的问题，不改变版本范围或下载物料。

### 8.2 swap查询：替换swaps及bad两行

在Reject swap generators and explicitly enabled swap units before changing swap任务的Python段里，保持与raw行一致的缩进，改为：

```python
    swaps = subprocess.check_output(['systemctl', 'list-unit-files', '--no-legend', '--no-pager', '--full'], text=True)
    bad += [r for r in swaps.splitlines() if len(r.split()) > 1 and r.split()[0].endswith('.swap') and r.split()[1] in ('enabled', 'enabled-runtime')]
```

systemd249按swap类型查询无匹配时会返回非零，原代码因此异常。修正后查询完整列表再筛选，仍拒绝明确启用的swap单元和zram生成器。若报告真实swap/zram项，再处理具体项，不能删除整个检查。

### 8.3 etcd验收：强制两个命令参数保持JSON字符串

原包直接把etcd的stdout传给ha_checks.py时，Ansible可能自动把JSON转成列表/字典，再用带单引号的Python表示传给命令。脚本使用json.loads，因此报`Expecting property name enclosed in double quotes`。这是验收传参错误，不能据此判定etcd故障；不能通过跳过三成员校验解决。[Ansible官方JSON过滤器说明](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/to_json_filter.html)

在新master1备份并编辑原任务文件：

```bash
mkdir -p work/manual-backup
cp -an playbooks/tasks/verify_ha.yml work/manual-backup/verify_ha.yml
vi playbooks/tasks/verify_ha.yml
```

找到`Validate one etcd cluster, exactly three voting members and a shared live leader`任务，将这个任务完整核对为下面内容。保持它在现有集群检查block中的层级，其他任务不改：

<!-- validate-etcd-task -->
```yaml
  - name: Validate one etcd cluster, exactly three voting members and a shared live leader
    ansible.builtin.command:
      argv:
        - '{{ ansible_playbook_python }}'
        - '{{ playbook_dir }}/../scripts/ha_checks.py'
        - etcd
        - '{{ ha_etcd_status.stdout | from_json | to_json }}'
        - '{{ ha_etcd_members.stdout | from_json | to_json }}'
        - "{{ groups['control_plane'] | to_json }}"
    delegate_to: localhost
    become: false
    changed_when: false
```

关键是两个stdout都先from_json再to_json，避免变成Python单引号表示或重复编码。保留严格JSON解析和三voter校验。site.yml、verify.yml和failover.yml共用此文件，只改这一处。

当前已经卡在该验收任务的机器，完成修改后直接进入第12节；从零部署则继续第9节。

## 9. Keepalived：在原工程中手工配置，一次配置三台

### 9.1 最终行为与配置位置

**在新master1修改工程模板和任务，再由Ansible部署到三台。** 不单独修改目标/etc配置，防止重跑site.yml时被模板覆盖。

采用以下行为：

| 节点 | 初始状态 | 优先级 | 抢占 | preempt_delay |
|---|---|---:|---|---|
| master1 | MASTER | 100 | 允许 | 不设置 |
| master2 | BACKUP | 50 | 允许 | 10秒 |
| master3 | BACKUP | 25 | 允许 | 10秒 |

不保留nopreempt。初始MASTER下preempt_delay不生效，因此master1恢复后可能直接抢回VIP；不能声称三台恢复都延迟10秒。[Keepalived 2.2.4官方说明](https://raw.githubusercontent.com/acassen/keepalived/v2.2.4/doc/man/man5/keepalived.conf.5.in)

单播、PASS、GARP刷新、HAProxy健康检查和四种状态通知均在下面完整配置中。自身网卡由Keepalived自动监控。两个显式零间隔参数和重复跟踪自身网卡的track_interface块都不加入模板。

### 9.2 创建两个脚本源文件

先在新master1工程创建目录，再分别编辑两个文件：

```bash
mkdir -p playbooks/files
vi playbooks/files/do_sth.sh
```

第一个文件完整内容：

<!-- validate-file: playbooks/files/do_sth.sh -->
```sh
#!/bin/sh
set -eu
exec /usr/bin/systemctl is-active --quiet haproxy
```

它检测HAProxy进程状态，不要求本地API已初始化；HAProxy可以转发到其他健康API节点，避免init前VIP一直不可用。

第二个文件：先执行`vi playbooks/files/notify_action.sh`，完整内容如下：

<!-- validate-file: playbooks/files/notify_action.sh -->
```sh
#!/bin/sh
set -eu
case "${1:-}" in
    MASTER|BACKUP|FAULT|STOP) ;;
    *) printf '%s\n' 'Expected MASTER, BACKUP, FAULT or STOP' >&2; exit 2 ;;
esac
exec /usr/bin/logger -t keepalived-notify -- "state=$1"
```

通知脚本仅记录本机状态，供journalctl查询。若veStack另有明确通知动作要求，以其实际文档为准；这里不包含云API操作。

### 9.3 替换原playbooks/templates/keepalived.conf.j2内容

先备份原文件，再编辑：

```bash
cp -an playbooks/templates/keepalived.conf.j2 work/manual-backup/keepalived.conf.j2
vi playbooks/templates/keepalived.conf.j2
```

将文件完整内容改为：

<!-- validate-file: playbooks/templates/keepalived.conf.j2 -->
```jinja2
global_defs {
    router_id K8S_{{ inventory_hostname }}
    vrrp_skip_check_adv_addr
    enable_script_security
    script_user root
}
vrrp_script checkhaproxy {
    script "/etc/keepalived/do_sth.sh"
    interval 5
    timeout 3
    weight 0
}
vrrp_instance VI_1 {
    state {{ 'MASTER' if inventory_hostname == groups['control_plane'][0] else 'BACKUP' }}
    interface {{ ha_effective_interface }}
    virtual_router_id {{ ha_router_id }}
    priority {{ ha_priorities[groups['control_plane'].index(inventory_hostname)] }}
    advert_int 1
    version 2
{% if inventory_hostname != groups['control_plane'][0] %}
    preempt_delay {{ ha_preempt_delay }}
{% endif %}
    authentication {
        auth_type PASS
        auth_pass {{ ha_auth_pass }}
    }
    unicast_src_ip {{ kube_node_ip }}
    unicast_peer {
{% for peer in ha_nodes if peer != kube_node_ip %}
        {{ peer }}
{% endfor %}
    }
    virtual_ipaddress {
        {{ ha_vip }}/{{ ha_vip_prefix }} dev {{ ha_effective_interface }}
    }
    notify_master "/etc/keepalived/notify_action.sh MASTER"
    notify_backup "/etc/keepalived/notify_action.sh BACKUP"
    notify_fault "/etc/keepalived/notify_action.sh FAULT"
    notify_stop "/etc/keepalived/notify_action.sh STOP"
    garp_master_delay 1
    garp_master_refresh 5
    track_script {
        checkhaproxy
    }
}
```

每台实际主IP由inventory提供；unicast_peer自动生成另外两台，不填VIP、不能只配一台peer。/32是云HaVIP的本机地址前缀，节点原有主IP网段不修改。

**Keepalived 2.2.4兼容性：** 不配置`vrrp_garp_interval 0`或`vrrp_gna_interval 0`，使用默认间隔；保留`garp_master_delay 1`和`garp_master_refresh 5`。这两个显式间隔参数的解析下限为0.000001秒，零值会使配置校验失败。[官方解析实现](https://raw.githubusercontent.com/acassen/keepalived/v2.2.4/keepalived/core/global_parser.c)

`interface {{ ha_effective_interface }}`已经确定并监控当前网卡，不能再把同一网卡加入`track_interface`；2.2.4将这种重复设置按配置错误处理。无需删除`interface`行，也不要删除HAProxy的`track_script`。[官方网卡处理实现](https://raw.githubusercontent.com/acassen/keepalived/v2.2.4/keepalived/vrrp/vrrp.c)

### 9.4 替换原playbooks/tasks/ha_lb.yml内容

此任务让site.yml在校验Keepalived配置前，先把两个脚本放到三台并设root权限。否则只有模板引用脚本、机器上却没有文件，会被原生校验拒绝或检查失效。

```bash
cp -an playbooks/tasks/ha_lb.yml work/manual-backup/ha_lb.yml
vi playbooks/tasks/ha_lb.yml
```

将完整内容改为：

<!-- validate-file: playbooks/tasks/ha_lb.yml -->
```yaml
---
- name: Validate the manual Keepalived settings
  ansible.builtin.assert:
    that:
      - ha_priorities == [100, 50, 25]
      - ha_auth_pass is string
      - ha_auth_pass is match('^[A-Za-z0-9]{1,8}$')
      - ha_preempt_delay | int >= 0
      - ha_preempt_delay | int <= 1000

- name: Install and validate the HAProxy TCP forwarding configuration
  ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    mode: '0644'
    validate: '/usr/sbin/haproxy -c -f %s'
  register: ha_proxy_config

- name: Create the keepalived configuration directory
  ansible.builtin.file:
    path: /etc/keepalived
    state: directory
    mode: '0755'

- name: Install the health and notification scripts
  ansible.builtin.copy:
    src: '{{ playbook_dir }}/files/{{ item }}'
    dest: '/etc/keepalived/{{ item }}'
    owner: root
    group: root
    mode: '0755'
    validate: '/bin/sh -n %s'
  loop: [do_sth.sh, notify_action.sh]
  register: ha_keepalived_scripts

- name: Install and validate cloud-compatible unicast VRRP configuration
  ansible.builtin.template:
    src: keepalived.conf.j2
    dest: /etc/keepalived/keepalived.conf
    mode: '0600'
    validate: '/usr/sbin/keepalived --config-test --use-file=%s'
  diff: false
  register: ha_keepalived_config

- name: Enable and start HAProxy
  ansible.builtin.systemd_service:
    name: haproxy
    enabled: true
    state: "{{ 'restarted' if ha_proxy_config.changed else 'started' }}"

- name: Require the local HAProxy frontend to listen
  ansible.builtin.wait_for:
    host: '{{ kube_node_ip }}'
    port: '{{ ha_api_port }}'
    timeout: 30

- name: Enable and start keepalived after HAProxy is running
  ansible.builtin.systemd_service:
    name: keepalived
    enabled: true
    state: "{{ 'restarted' if (ha_keepalived_config.changed or ha_keepalived_scripts.changed) else 'started' }}"
```

HAProxy模板保持原包内容：监听0.0.0.0:8443，TCP转发三台主IP:6443，并做TCP健康检查。不改成直接占用6443，否则会与本机kube-apiserver冲突。

### 9.5 让两个新脚本纳入原工程的备份/回滚

首次site部署前，在`config/runtime.yml`的managed_paths列表末尾补两项：

```yaml
- /etc/keepalived/do_sth.sh
- /etc/keepalived/notify_action.sh
```

同时编辑`scripts/rollback_node.py`，在文件顶部`ALLOWED = { ... }`集合里补两项（注意引号和逗号）：

```python
    '/etc/keepalived/do_sth.sh', '/etc/keepalived/notify_action.sh',
```

两处需要配套；只加managed_paths不加ALLOWED会使原回滚拒绝处理。首次备份在脚本部署之前创建，因此新系统上不存在的脚本会在回滚时删除，原来存在的文件则按首次备份恢复。不要等site已经备份后再补来假装基线已包含。

已在当前这轮部署完成以上修改时，仅核对最终内容，不重新创建备份基线。

## 10. 用原包建立独立预检查，先检查再安装

原包没有预检查标签。在m1按下面内容创建`playbooks/preflight-manual.yml`，它只导入现有预检查任务，不导入安装任务：

```bash
vi playbooks/preflight-manual.yml
```

完整内容：

<!-- validate-file: playbooks/preflight-manual.yml -->
```yaml
---
- name: Check all three nodes before installation
  hosts: control_plane
  become: true
  gather_facts: true
  any_errors_fatal: true
  environment:
    PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    HTTP_PROXY: ''
    HTTPS_PROXY: ''
    ALL_PROXY: ''
    http_proxy: ''
    https_proxy: ''
    all_proxy: ''
    NO_PROXY: '*'
    no_proxy: '*'
  vars_files:
    - ../config/cluster.yml
    - ../config/runtime.yml
    - ../artifacts/lock.yml
  tasks:
    - name: Run the existing preflight checks only
      ansible.builtin.import_tasks: tasks/preflight.yml
    - name: Require the common internal interface and valid Keepalived authentication
      ansible.builtin.assert:
        that:
          - flannel_iface | default('') | length > 0
          - flannel_iface == ha_effective_interface
          - ha_auth_pass is string
          - ha_auth_pass is match('^[A-Za-z0-9]{1,8}$')
        fail_msg: Use the common real internal Flannel interface and a quoted 1-8 character alphanumeric ha_auth_pass.
```

在m1做语法检查及真正预检查：

```bash
sh -n playbooks/files/do_sth.sh
sh -n playbooks/files/notify_action.sh
python3 -m py_compile scripts/rollback_node.py
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/site.yml --syntax-check
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/preflight-manual.yml --syntax-check
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/preflight-manual.yml
```

成功标准：三台failed=0、unreachable=0。检查不改系统配置，仅有Ansible临时文件；不使用site.yml --check替代。若提示Missing工具并让你执行bootstrap，不照旧提示全量安装，只补对应兼容工具。

普通fstab swap会在安装时关闭；明确启用的swap单元/zram生成器需要单独处理，防止重启重新开启。预检查失败先修具体项，不删除断言。

正式安装后工程会检查二进制版本与CRI。出现GLIBC缺失或Illegal instruction时先停下选兼容构建，不替换系统libc；原生二进制能启动不等于CRI/CNI已验收。

## 11. 运行原site.yml，完成三主集群

前提：上面所有步骤通过，三台都是本次新目标机器；配置已确定且Keepalived模板/任务配套修改完成。

```bash
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/site.yml
```

原包按如下顺序自动执行，不需要分别对三台运行kubeadm init：

1. 三台完整预检查、校验物料并复制。
2. 保存首次主机配置备份与集群指纹。
3. 关闭swap、注释fstab swap项，加载overlay/br_netfilter，配置转发sysctl。
4. 安装containerd/runc/CRI/CNI/Kubernetes二进制；containerd与kubelet使用systemd cgroup，pause与lock一致。
5. 镜像导入containerd的k8s.io namespace，保留原始引用。
6. 三台部署并原生校验HAProxy/Keepalived，启动并开机自启；检查HaVIP只有一个owner。
7. 仅master1 kubeadm init，稳定API为172.16.0.4:8443；应用本地Flannel/CoreDNS。
8. master2、master3以serial:1依次加入控制平面与etcd，前一台自身API ready后才加入下一台。
9. 清理本轮临时token/共享材料，验证三节点、etcd与跨节点网络。

安装中会复制整个Deb物料目录，但不会安装其中的OS Deb。网络镜像/控制平面镜像使用Never，不在线补拉。三台角色显示control-plane仍能运行业务Pod，不需要再出现worker字样。

Keepalived只有一个VIP owner；在API尚未初始化之前，HAProxy前端TCP可连接并不代表API ready。kubelet在init/join配置尚未生成前反复重启，也不单独证明安装失败。

首次site.yml会继续完成验收。若只在验收阶段失败，按具体报错修正后使用第12节独立verify.yml，不为此重跑安装或reset。

## 12. 正常验收与继续当前验收

在新master1执行。**你目前已走到etcd验收，确认第8.3节参数修正后从这里继续。** 先语法检查，再实际验收：

```bash
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/verify.yml --syntax-check
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/verify.yml
/usr/local/bin/kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes -o wide
/usr/local/bin/kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -A -o wide
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'systemctl is-active containerd kubelet haproxy keepalived'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'systemctl is-enabled containerd kubelet haproxy keepalived'
```

必须同时满足：恰好三个预期Node Ready、三个直连API与VIP正常、CRI RuntimeReady/NetworkReady、四服务active+enabled、三个同集群etcd voter健康且共享leader、Flannel/kube-proxy/CoreDNS正常、跨节点Pod/Service/DNS实际测试通过。只看Node Ready不足以验收。m1成功记录在`/var/lib/k8s-offline/verification.json`。

### 12.1 Keepalived现场核对

```bash
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'keepalived --config-test --use-file=/etc/keepalived/keepalived.conf'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'ip -o -4 address show scope global'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'journalctl -u keepalived --since=-15min --no-pager'
env -u PYTHONPATH .venv/bin/ansible -i inventory.ini control_plane -b -m ansible.builtin.command -a 'journalctl -t keepalived-notify --since=-15min --no-pager'
```

三台原生配置检查成功，VIP只在一台主机上；必要时分别查询keepalived服务日志和keepalived-notify标记日志。核对MASTER/BACKUP/BACKUP配置、100/50/25、无nopreempt、三台同PASS、各自源IP/另外两台peer、正确网卡和VIP。auth_pass属于敏感配置，不原样分享完整配置。

### 12.2 如何判断完成

| 检查 | 通过条件 |
|---|---|
| Node/API | 恰好master1/2/3，全部Ready；三台:6443和HaVIP:8443 ready |
| etcd | 恰好三个voting member，同一cluster_id、有效共同leader、无learner，三endpoint health通过 |
| 运行环境 | 三台CRI RuntimeReady/NetworkReady为true，四项服务均active+enabled |
| 网络 | Flannel、kube-proxy、CoreDNS rollout正常；跨节点Pod IP、ClusterIP、DNS smoke通过 |
| 结果 | 三台recap均failed=0、unreachable=0；m1生成verification.json |

集群级检查只在master1执行，因此master2/master3对这些任务显示skipping是正常现象。单个任务ok、相同leader或Node Ready均不能代替全部验收。验收临时创建的测试namespace会在测试结束清理。

## 13. 故障切换与逐台重启

正常验收通过后，在可接受短暂API切换的时段执行：

```bash
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/failover.yml
```

它停当前owner的HAProxy，要求VIP迁移、API恢复，并在always中恢复原HAProxy。成功记录`/var/lib/k8s-offline/failover-verified.json`。抢占模式下master1恢复后可能抢回，不能要求VIP永久留在备用机。

再通过控制台一次只重启一台，每台回来后运行verify.yml，三etcd恢复健康才重启下一台。三成员etcd只容忍一台故障，不能同时重启两台。进程切换通过不等于云断电、网络分区、存储故障都已验收。

## 14. 常见报错与继续位置

| 报错/现象 | 处理 |
|---|---|
| manifest提示Artifact checksums...verified | 成功，继续打包/传输，不是报错 |
| package.sh提示Output exists | 校验现有完整包，不覆盖 |
| ha_auth_pass is string失败 | 第5.2节保留英文引号、去掉重复键，再检查类型 |
| vrrp_garp_interval/vrrp_gna_interval的0非法 | 核对第9.3节最终模板，不添加两个零间隔参数 |
| Ignoring track_interface ... since own interface | 删除重复跟踪自身网卡的块，保留interface行和track_script |
| HA validation failed: Expecting property name enclosed in double quotes | 第8.3节修正两个JSON参数，只运行第12节verify.yml |
| active:ufw但ufw显示inactive | 第6节核对真实规则与云安全组后填写确认项 |
| MISSING某工具/依赖冲突 | 第4/7节只选必要兼容Deb，先模拟，不全量bootstrap |
| .venv/bin/python: No module named pip | 第4.3节的安装命令需要临时PYTHONPATH；以Ansible可运行为准 |
| admin.conf与完成marker不一致 | 保护性停止，保留现场，不删marker/fingerprint强行继续 |
| 指纹不一致 | 核对原本的IP/VIP/节点顺序/网段/lock，不直接改身份或删状态 |
| ImagePullBackOff / Never缺镜像 | 核对原引用、lock、containerd k8s.io namespace，不临时改成联网拉取 |
| Flannel/跨节点不通 | 查共同网卡、Pod CIDR、UDP8472及主机转发规则 |

### 14.1 join失败的查询

原包join任务使用no_log保护token与PKI；censored表示结果隐藏。原包不会自动生成join-result文件。先登录失败节点只查询日志：

```bash
sudo journalctl -u kubelet -u containerd --since=-30min --no-pager -n 300
sudo crictl ps -a
sudo ss -lntp
sudo ls -l /etc/kubernetes /var/lib/k8s-offline
```

根据crictl ps -a的真实容器ID查询对应etcd/kube-apiserver日志：

```bash
# 将CONTAINER_ID替换为实际容器ID。
sudo crictl logs CONTAINER_ID
```

检查主IP双向2379/2380、VIP8443→节点6443、VRRP112、时间与证书、镜像引用。master2失败会阻止master3继续join，不表示master3本身已失败。分享错误前去除token/私钥，不关闭no_log公开全部凭据。

仅预检查失败时，修正前提后可以重跑。已成功且状态一致的init/join会跳过；部分init/join失败时先诊断，不保证能直接安全重跑，更不能通过删除完成标记伪造成功。

## 15. 基础回滚与数据边界

尚未部署的三台新机器不需要先回滚。只有完成本次部署后又明确决定拆除整个本工程集群时，先在外部备份所需数据，再执行下面破坏性命令：

```bash
env -u PYTHONPATH .venv/bin/ansible-playbook -i inventory.ini playbooks/rollback.yml \
  -e rollback_confirm=true -e destroy_cluster_data=true
```

要求三台都在inventory目标中，不能--limit单台。删除本地etcd/PKI/Pod/运行时状态并恢复首次备份覆盖的主机配置；第9.5节配套修改后两个脚本也纳入恢复。首次备份发生在最小依赖装好之后，因此不卸载OS依赖、不恢复云出厂系统、不删除云HaVIP、不恢复业务卷。

回滚后按输出重启，保留备份，并按原工程运维说明归档整个/var/lib/k8s-offline、建立新备份基线后才能重建。该入口不是单个etcd成员替换方案，不把回滚当所有安装失败的通用修复。

## 16. 修改文件清单与验证边界

### 16.1 本手册在原工程中修改/新增的内容

| 文件 | 作用 |
|---|---|
| inventory.ini、config/cluster.yml | 三台新节点、HaVIP、网段、网卡、认证及规则确认 |
| playbooks/tasks/preflight.yml | 版本正则和systemd249 swap查询两处修正 |
| playbooks/tasks/verify_ha.yml | etcd状态/成员参数显式JSON序列化 |
| playbooks/templates/keepalived.conf.j2 | 最终三节点Keepalived配置 |
| playbooks/files/do_sth.sh、notify_action.sh | HAProxy健康检查与本地状态通知 |
| playbooks/tasks/ha_lb.yml | 先部署脚本、再原生校验配置，启动HA服务 |
| config/runtime.yml、scripts/rollback_node.py | 两个脚本的备份/回滚路径配套 |
| playbooks/preflight-manual.yml | 独立检查入口，不执行安装 |

下载物料、config/versions.env、config/downloads.env和artifacts/lock.yml保持与准备机成功包一致。手工修改后SOURCE_FILES.sha256不再等于初始源码快照属于预期；不修改这个快照假装未编辑，artifacts独立校验必须继续通过。

### 16.2 交付及实机边界

原源码包SHA256：

```text
82f8323bdc1ba3ed96346d8905b6ead05f1fd7ac6510eab7c835ed798627daed
```

本次仅交付文档，不生成新源码/补丁包，不操作你的服务器。本地检查使用Python 3.13/Ansible 2.18，覆盖命令语法、手工YAML、原工程入口的Ansible语法、三份Keepalived模板和备份/回滚路径；实际目标Python 3.10/Ansible 2.17.14与原生服务需现场验收。

etcd JSON问题已在本地重现，修正后包含64位ID的样例通过；learner、不同集群ID、不同leader样例仍被拒绝。这个检查不等于远程验证你的etcd。最近现场日志已进入etcd验收，但尚未收到最终整体验收成功记录。

本手册适用前述Ubuntu 22.04 amd64系统基线，不能保证同一物料覆盖任意精简镜像或其他全新Linux发行版。离线业务镜像、应用副本、持久卷、Ingress、监控和日常数据备份需按业务另行配置。
