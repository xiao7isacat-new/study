# linux
## 服务器
### 查看sn
    dmidecode -s system-serial-number
### 查看制造商
    dmidecode -t system
    dmidecode -s system-product-name
### 查看内存信息
    dmidecode -t memory
    dmidecode -t memory | grep -E "Type:|Speed:|Size"

### 磁盘信息
    nvme list
    lsblk -o NAME,HCTL,SIZE,MODEL,SERIAL
    lsblk -d -o NAME,MODEL,SERIAL,REV,SIZE,TYPE
    for i in /sys/class/nvme/nvme*; do echo -n "$(basename $i): "; readlink -f $i; done
    ls -l /sys/class/nvme/*/device
    lspci  | grep Non-Vol

### cpu
    查看cpu的命令: nproc

### mlx
    mst status -v
    mlxlink -d 65:01.0 -m
    mlxlink -d  65:01.0 -c

### pci
    lspci -vvs `ethtool -i eth1|grep bus|awk -F '0000:' '{print $2}'`|grep PN

### pcie Current
    nvidia-smi -q | grep -A 5 "PCIe"
    nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current,pcie.link.width.max --format=csv pcie.link.gen.current, pcie.link.gen.max, pcie.link.width.current, pcie.link.width.max
    ./nvbandwidth --testcase device_to_host_memcpy_ce host_to_device_memcpy_ce device_to_device_memcpy_read_ce device_to_device_memcpy_write_ce all_to_host_memcpy_ce host_to_all_memcpy_ce -i 5





## 网络
### arp
    arp -n | grep 10.204.91.59
    arp -d 10.204.91.59
    arp -s 10.204.91.59 8c:91:3a:4b:6e:9b

### 查看网卡版本
    mlxfwmanager
### ib
    iblinkinfo | grep -B2 -A2 -i "$(hostname)"


## cmd
### 证书
    解析证书
    openssl x509 -in server-cert.pem -noout -text

### 添加引号，用逗号分隔
    sed 's/.*/"&"/' test.txt | tr '\n' ',' | sed 's/,$//'
### 校验脚本包完整性
    echo "f1ba01553b26944c247a73dc0f4a3288  /tmp/cfs-upgrade-1.6.3-to-local-config.tar.gz" | md5sum -c -
### 文件大小
    du -sh ./* | sort -hr

### 进程
    ps -eo pid,ppid,stat,comm,args | awk 'NR==1 || $3 ~ /^Z/'

### 网卡
    ethtool -S eth3 | grep -iE 'err|crc|drop|fault|disc|down|link|fec|symbol'
### gobgp
    gobgp neighbor


## git
    # 1. 大幅提高 HTTP 缓冲区（默认仅 1MB，对大仓库远远不够）
    git config --global http.postBuffer 524288000
    
    # 2. 放宽低速判定，避免代理或网络波动导致 Git 主动断开
    git config --global http.lowSpeedLimit 0
    git config --global http.lowSpeedTime 999999
    
    # 3. 关闭压缩，减少接收端解包时的内存消耗，避免 index-pack 错误
    git config --global core.compression 0
    
    # 4. 强制使用 HTTP/1.1，有时 HTTP/2 在代理下不稳定
    git config --global http.version HTTP/1.1

    # 浅克隆
    git fetch --depth 1 origin release-1.32:release-1.32
    # 显式地从远程拉取该分支的最新信息
    git fetch --depth 1 origin release-1.32:release-1.32
    git checkout release-1.32

    # 配置代理
    git config --global https.proxy  socks5://172.16.0.41:8080
    





# sql
## mysql
### mysqldump
    mysqldump -h iaas.vestack.cloud -P 33100 -u user_admin -p'*qwer@k8s!_PL' --max_allowed_packet=1G  vecloud_flow param > vecloud_flow_param.sql
    mysqldump -h iaas.vestack.cloud -P 24001 -u user_admin   -p'*qwer@k8s!_PL' --single-transaction --skip-lock-tables --routines  --triggers --events  --hex-blob --default-character-set=utf8mb4 --databases cetus  > cetus_full_$(date +%Y%m%d_%H%M%S).sql



### 查看json 中的内容
    SELECT *
    FROM instances
    WHERE metadata IS NOT NULL
      AND metadata <> ''
      AND JSON_VALID(metadata)
      AND JSON_SEARCH(metadata, 'one', '9.234.128.102') IS NOT NULL\G;
## redis
### redis-dump
    redis-dump -H 127.0.0.1 -d 10 > redis_db10_$(date +%Y%m%d_%H%M%S).json



# server
## dhcp

### client
    cat /var/lib/dhcp/dhclient.leases
### server
    cat /var/lib/dhcp/dhcpd.leases | grep -A 15  26.4.0.194

## chrony
    chronyc tracking
    chronyc sources -v

## nexus
    gluster volume info gv0
    gluster volume status gv0
    gluster volume status gv0
    gluster volume heal  gv0 info

# switch
## h3c
    display bgp instance DCONBGP peer ipv4 vpn-instance-all
    display ip interface brief

# ansible
## 常用命令
    批量升级网卡固件 并发执行
     ansible all -i fireword_updata -u root -f 24 -m shell -a 'cd /root && flint -d ca:00.0 -i fw-ConnectX6Dx-rel-22_35_3502-MCX623106AN-CDA_Ax-UEFI-14.29.15-FlexBoot-3.6.902.bin -y burn && systemd-run --on-active=5s "$(command -v ipmitool)" chassis power cycle' | tee nic_fw_upgrade.log


# 虚拟机
## cpu
    查看当前所有 QEMU/KVM 虚拟机合计的实际 CPU 使用率：
    HOST_CPUS=$(nproc)
    
    ps -eo comm=,pcpu= | awk '
      $1 ~ /^(qemu-kvm|qemu-system)/ { sum += $2 }
      END { printf "VM 合计: %.2f%%，约占 %.2f 个逻辑 CPU\n", sum, sum / 100 }
    '
    
    ps -eo comm=,pcpu= | awk -v n="$HOST_CPUS" '
      $1 ~ /^(qemu-kvm|qemu-system)/ { sum += $2 }
      END { printf "占宿主机全部 %d 个逻辑 CPU 的 %.2f%%\n", n, sum / n }
    '
    
    查看所有运行中 VM 配置的 vCPU 总数（注意：这是配置量，不是实际占用）：
    virsh list --name | while IFS= read -r vm; do
      [ -n "$vm" ] || continue
      virsh dominfo "$vm" | awk -F: -v vm="$vm" '
        /^CPU\(s\)/ { gsub(/^[ \t]+/, "", $2); print vm, $2 }'
    done | awk '{sum += $2; print $0} END {print "配置 vCPU 合计:", sum}'




# 容器
## docker
    docker images --tree
    docker image inspect xxx:x

## k8s

    kubectl scale deploy   -n cfs-cloudfs-sfcs cfs-nn-http-proxy-cloudfs-sfcs  --replicas=3
    
    kubectl get pv mymeta-266458c99a09-data-1 -o json | jq '.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions'
    
    kubectl get pv mymeta-266458c99a09-data-0 -o json | jq '.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions'
    
    kubectl get pod -A -o wide | awk '$8=="iaas-ctl-k8s-w1"'

## 查看容器的名称空间
    CID=<上一步查到的容器ID>
    
    PID=$(crictl inspect "$CID" | jq -r '.info.pid // .status.pid')
    
    cat /proc/"$PID"/cgroup


## 查看资源使用率高的容器
    crictl stats --all
    
    for cid in \
      3e06856dd3496 \
      345e75da5db87 \
      923928e613158 \
      02c2442d672cb \
      832c8684ec593 \
      748c39a214fe0
    do
      crictl inspect "$cid" | jq -r '
        [
          .status.id[0:12],
          .status.labels["io.kubernetes.pod.namespace"],
          .status.labels["io.kubernetes.pod.name"],
          .status.labels["io.kubernetes.container.name"]
        ] | @tsv'
    done


## 查看历史版本
kubectl get rs -n volc -o json | jq -r '
  .items[]
  | select(.metadata.name | startswith("cloud-fileserver-"))
  | [
      .metadata.name,
      (.metadata.annotations["deployment.kubernetes.io/revision"] // "-"),
      .metadata.creationTimestamp,
      (.spec.template.spec.containers
        | map(.name + "=" + .image)
        | join(","))
    ]
  | @tsv
' | column -t -s $'\t'


# 业务相关
## ebs
## 后台扩容
    nohup env PRODUCT_LINE=storage hyctl flow nodepool -c ebs_intel_pool-c.yaml > ebs_intel_pool-c.log 2>&1 &
## 获取所有的master信息
    curl -s 127.0.0.1:32784/clusters | grep master | grep BM

## uniagent
    MySQL [uniagent_svr]> select * from component_host where deploy_status="exception";

## vecloud_flow
    select id,task_status from task_execution_record where workflow_id="11454" and execute_id in ('2091902507268894720','2091902507906428928','2091902511619067904','2091902512575369216','2091902512793473024','2091902512940273664','2091902513023479808','2091902513459687424','2091902514802544640');

    update task_execution_record set task_status="failed" where id="xx";




# code


## go
### install
    curl -x socks5h://172.16.0.41:8080 -L -O https://go.dev/dl/go1.27.0.linux-amd64.tar.gz
    sudo tar -C /usr/local -xzf go1.27.0.linux-amd64.tar.gz
     cat >> ~/.bashrc <<'EOF'
    export GOROOT=/usr/local/go
    export GOPATH=$HOME/go
    export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
    EOF   # 将 Go 的路径及 GOPATH 配置追加到 ~/.bashrc
    
    # 使配置立即生效
    source ~/.bashrc
    
    
    go env -w GO111MODULE=on
    go env -w GOPROXY=https://goproxy.cn,direct
