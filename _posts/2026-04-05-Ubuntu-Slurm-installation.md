---
layout: post
title: "Ubuntu—VASP installation"
date: 05/04/2026
categories: work-notes
tags: linux, ubuntu, vasp, slurm
---

## Configurations

1. CPU: Intel Xeon E5-2696 v4
2. Socket: 2
3. Cores: 44
4. CPUs: 88
5. Flags: AVX2, FMA3
6. RAM: 64GB
7. SSD: 500GB
8. Ubuntu: 24.04.4

## Munge and Slurm installation

1. 老规矩，先更新系统包列表

```bash
sudo apt update
```

2. 安装 munge 和 slurm 管理组件

```bash
sudo apt install munge slurm-wlm -y
```

3. 生成 munge 密钥

```bash
sudo /usr/sbin/mungekey -f
sudo chown munge: /etc/munge/munge.key 
sudo chmod 400 /etc/munge/munge.key
```
> note: 锁紧权限，确保只有 munge 进程能读它

4. 启动并使 munge 开机自启

```bash
sudo systemctl enable munge
sudo systemctl restart munge
```

## 配置 Slurm (slurm.conf)

1. 给自己slurm文件夹下的临时权限

```bash
sudo chown -R $USER:$USER /etc/slurm
```

2. 创建 `slurm.conf` 并写入配置

```bash
# 禁用邮件程序（消除警告）
MailProg=/usr/bin/true

# 集群名称与控制节点
ClusterName=JQL_Cluster # 自定义
SlurmctldHost=JQL24    #hostname一下，运行“管理进程（slurmctld）”的主机名

# 调度与选择策略（单机环境不用修改）
SlurmUser=slurm
SlurmdUser=root
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
StateSaveLocation=/var/lib/slurm/slurmctld
SlurmdSpoolDir=/var/lib/slurm/slurmd
SwitchType=switch/none
MpiDefault=none

# 资源分配策略（单机环境不用修改）
ProctrackType=proctrack/cgroup
ReturnToService=1
TaskPlugin=task/affinity,task/cgroup

# 调度器设置（单机环境不用修改）
SchedulerType=sched/backfill
SelectType=select/cons_tres # 之前是cons_res就报错了
SelectTypeParameters=CR_Core_Memory

# 日志与进程（单机环境不用修改）
SlurmctldPidFile=/run/slurmctld.pid
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdPidFile=/run/slurmd.pid
SlurmdLogFile=/var/log/slurm/slurmd.log

# --- 节点与队列配置 ---
# 核心设置：CPUs=44，下面的设置必须和单机的物理硬件匹配！
NodeName=JQL24 CPUs=44 State=UNKNOWN
PartitionName=debug Nodes=JQL24 Default=YES MaxTime=INFINITE State=UP
```
> note: 保持一个默认队列，单机够用了。

3. 锁紧权限

```bash
sudo chown root:root /etc/slurm/slurm.conf # 将所有权交还给 root
sudo chmod 644 /etc/slurm/slurm.conf # 设置权限为：root可读写，其他人只读 (644)
```

## 配置 Cgroup（防止 CPU 越权）

1. 创建 `cgroup.conf`，Slurm 默认使用 cgroup 来限制任务的 CPU 使用

```bash
CgroupAutomount=yes
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=no
```
> note: 参数详解：
>
> |参数|通俗解释|为什么要开启？|
> |---|---|---|
> |`CgroupAutomount=yes`|自动挂载控制组|让 Slurm 能够自动接管操作系统的资源控制接口，省去手动配置 Linux 内核参数的麻烦。|
> |**`ConstrainCores=yes`**|**核心隔离 (最重要)**|如果在脚本里写 `#SBATCH -n 20`，Slurm 会通过物理手段把这 20 个核“锁”给这个任务。即便程序代码写错了想多占核，系统也会拒绝。|
> |`ConstrainDevices=yes`|设备隔离|限制任务对硬件设备（如显卡 GPU）的访问。确保没申请 GPU 的任务看不见也用不了 GPU。|
> |`ConstrainRAMSpace=yes`|**内存封顶**|防止某个 VASP 任务因为内存泄漏或体系太大而吃光所有内存，导致整台工作站（包括你的图形界面）直接卡死死机。|
> |`ConstrainSwapSpace=no`|不限制交换分区|虚拟内存（Swap）通常速度极慢，对于高性能计算，通常不限制它，或者干脆在系统层面关闭 Swap。|

## 创建目录并启动服务

1. 创建日志和状态文件夹

```bash
sudo mkdir -p /var/lib/slurm/slurmctld /var/lib/slurm/slurmd /var/log/slurm 
sudo chown -R slurm:slurm /var/lib/slurm /var/log/slurm
```
> note: `slurmctld` (Controller)：它是大脑。它负责读取 `slurm.conf`，知道整个集群有多少个队列、每个队列叫什么名字、优先级是多少。
> `slurmd` (Daemon)：它是手脚。它只负责在具体的计算节点上干活（启动进程、监控内存）。它并不关心自己属于哪个队列，它只关心“大脑”给它发了什么指令。

2. 启动服务

```bash
sudo systemctl enable slurmctld slurmd
sudo systemctl restart slurmctld slurmd

# 如果报错，根据错误重新修改对应的conf文件，然后执行
sudo systemctl stop slurmctld slurmd # 停止服务
sudo systemctl start slurmctld # 重新启动
sudo systemctl start slurmd # 重新启动
```

3. 检查状态

```bash
sinfo # 应显示 JQL24 为 idle 状态
```
> note: 如果节点状态显示 `down` 或 `drain`，执行 `sudo scontrol update nodename=JQL24 state=resume` 激活它。