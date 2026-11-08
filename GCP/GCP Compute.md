# Google Cloud Compute 知识体系

## 1. Persistent Disk 与磁盘扩容

### 1.1 Persistent Disk 类型

Google Cloud 提供多种 Block Storage，适用于不同性能、成本和工作负载需求。

|磁盘类型|特点|典型场景|
|---|---|---|
|Standard Persistent Disk（pd-standard）|HDD，成本较低，随机 IOPS 较低|顺序读写、冷数据、成本敏感型工作负载|
|Balanced Persistent Disk（pd-balanced）|SSD，性能与成本平衡|通用 VM、企业应用|
|SSD Persistent Disk（pd-ssd）|SSD，较高 IOPS 和吞吐量|数据库、低延迟应用|
|Extreme Persistent Disk（pd-extreme）|高性能 SSD，可单独配置 IOPS|高性能数据库、对 IOPS 要求很高的工作负载|
|Hyperdisk|新一代 Block Storage，可根据类型配置性能、容量等|高性能数据库、企业级工作负载|
|Local SSD|物理附加在宿主机上的本地 SSD，延迟低、性能高|临时缓存、Scratch Data、高性能临时存储|

**重点区别：**

- Standard、Balanced、SSD Persistent Disk 的性能通常与磁盘容量和磁盘类型相关。
    
- Extreme Persistent Disk 可以独立配置 IOPS，不需要单纯通过扩大容量来提升 IOPS。
    
- Hyperdisk 提供更灵活的性能配置，但具体能力取决于 Hyperdisk 类型。
    
- Local SSD 是临时性本地存储，不适合存放唯一副本的数据。
    

### 1.2 Local SSD

Local SSD 的特点：

- 直接连接到 VM 所在宿主机。
    
- 提供高 IOPS 和低延迟。
    
- 适合临时数据、缓存和临时计算文件。
    
- 不具备 Persistent Disk 相同的持久化保障。
    
- VM 停止、删除或发生宿主机故障时，数据可能丢失，具体行为取决于实例和 Local SSD 配置。
    

因此，Local SSD 不适合作为数据库唯一的数据存储位置。

### 1.3 Disk Expansion

Persistent Disk 和 Hyperdisk 通常支持在线扩容，不需要为了扩大磁盘容量而重启 VM。

基本流程：

```text
Increase Disk Size in GCP
          |
          v
Resize Partition (if needed)
          |
          v
Resize Filesystem
          |
          v
Filesystem Uses New Capacity
```

Linux 常见命令：

```bash
# 查看磁盘和分区
lsblk

# 查看文件系统
df -h

# ext4 文件系统扩容
sudo resize2fs /dev/sdb1

# XFS 文件系统扩容
sudo xfs_growfs /mount/point
```

注意：

1. 需要先在 Google Cloud 控制台或 gcloud 中扩大磁盘容量。
    
2. 如果使用分区，需要先扩大分区，再扩大文件系统。
    
3. `resize2fs` 用于 ext2/ext3/ext4 文件系统，不适用于 XFS。
    
4. XFS 使用 `xfs_growfs` 扩容。
    
5. 文件系统扩容通常可以在线完成。
    
6. Persistent Disk 通常支持扩容，但不支持直接缩小容量；如果需要更小的磁盘，需要迁移数据到新磁盘。
    

### 1.4 磁盘性能与容量的关系

对于部分 Persistent Disk 类型，IOPS 和吞吐量会受到磁盘容量、VM 机器类型以及磁盘类型等因素影响。

因此：

- 增加磁盘容量可能提高可获得的 IOPS 或吞吐量。
    
- 但最终性能也可能受到 VM 的磁盘性能上限限制。
    
- Extreme PD 可以单独配置 IOPS。
    
- Hyperdisk 的性能配置方式取决于具体产品类型。
    

不能简单认为所有磁盘都是容量越大，性能就一定线性增加。

## 2. Cloud Shell 持久化存储

Cloud Shell 并不是一个长期运行的普通 Compute Engine VM，而是 Google 管理的临时开发环境。

### 2.1 存储结构

Cloud Shell 为每个用户提供 5 GB 的持久化磁盘空间，挂载在：

```
$HOME
```

|位置|持久化情况|
|---|---|
|`$HOME`|跨 Cloud Shell Session 持久化|
|`$HOME` 下的脚本、配置文件、安装的软件|持久化|
|`$HOME` 以外的临时文件|通常随 Session VM 销毁而丢失|
|Ephemeral Mode 下的文件|Session 结束后全部丢失|

特点：

- `$HOME` 的存储按用户分配，而不是按 Project 分配。
    
- 同一个用户可以在不同 Project 中使用这份 Home 存储。
    
- Cloud Shell 的 VM 本身是临时的，Session 结束后可以被销毁。
    
- 长时间不使用 Cloud Shell，持久化 Home 存储也可能被回收；官方文档说明，连续 120 天未访问可能触发删除。
    

面试问题：

1. Cloud Shell 的 VM 和 `$HOME` 目录有什么区别？
    
2. Cloud Shell 中哪些文件可以跨 Session 保存？
    
3. Cloud Shell 的 5 GB 存储是按 Project 还是按 User 分配？
    
4. 如果需要长期运行、可自定义配置的开发环境，Cloud Shell 是否合适？
    

## 3. App Engine Standard vs Flexible

App Engine 是 Google Cloud 的 PaaS 服务，开发人员可以部署应用程序，而不需要直接管理底层 VM。

### 3.1 App Engine Standard Environment

特点：

- 应用运行在 Google 提供的 Sandbox 环境中。
    
- 使用受支持的语言 Runtime。
    
- 对系统调用、文件访问和运行环境有一定限制。
    
- 实例启动速度快。
    
- 支持快速自动扩缩容。
    
- 可以 Scale to Zero，即没有流量时缩容到 0 个实例。
    

适用场景：

- Stateless Web Application。
    
- 流量波动较大的应用。
    
- 希望降低基础设施管理成本的应用。
    

### 3.2 App Engine Flexible Environment

特点：

- 应用运行在 Compute Engine VM 上的 Docker Container 中。
    
- 可以使用自定义 Runtime 和 Dockerfile。
    
- 对系统库、运行环境和后台进程的限制更少。
    
- 可以选择 VM 的 CPU 和 Memory 配置。
    
- 实例启动和扩缩容通常比 Standard 慢。
    
- 每个活跃版本至少需要一个实例，不能 Scale to Zero。
    

适用场景：

- 需要自定义系统依赖的应用。
    
- 需要运行后台进程的应用。
    
- 需要更灵活控制 CPU 和 Memory 的应用。
    

### 3.3 对比

|对比|Standard|Flexible|
|---|---|---|
|运行环境|Sandbox|Docker Container|
|底层基础设施|Google 管理的实例环境|Compute Engine VM|
|自定义 Runtime|受限|更灵活|
|Scale to Zero|支持|不支持|
|启动速度|秒级|通常分钟级|
|扩缩容|快速|相对较慢|
|本地磁盘|主要使用临时 `/tmp`|临时 VM 磁盘|
|后台进程|有限制|支持|

面试问题：

1. App Engine Standard 和 Flexible 有什么区别？
    
2. 哪种 App Engine Environment 支持 Scale to Zero？
    
3. 为什么 Flexible Environment 不能 Scale to Zero？
    
4. 如果应用需要自定义 Docker Image 和系统依赖，应该选择哪种环境？
    
5. App Engine 和 Compute Engine 的主要区别是什么？
    

## 4. Disk Snapshot、Image、Machine Image 与 Instance Template

这几个概念都可以用于 VM 部署、复制、备份和恢复，但保存的内容不同。

### 4.1 Disk Snapshot

Snapshot 是磁盘在某个时间点的数据备份。

特点：

- 保存磁盘数据的时间点状态。
    
- 支持增量存储，后续 Snapshot 通常只保存变化的数据块。
    
- 可以用于创建新的 Persistent Disk。
    
- 可以用于数据恢复。
    
- 不保存完整的 VM 配置，例如 VM 的 Machine Type、网络配置等。
    

适用场景：

- 磁盘备份。
    
- 磁盘恢复。
    
- 创建新磁盘。
    
- 灾难恢复。
    

### 4.2 Custom Image

Custom Image 是用于创建新 Boot Disk 或 VM 的磁盘镜像。

通常包含：

- 操作系统。
    
- 已安装的软件。
    
- 系统配置。
    
- 磁盘文件系统和数据。
    

适用场景：

- 创建标准化 VM。
    
- 批量部署相同的操作系统和软件环境。
    
- 制作自定义操作系统镜像。
    

### 4.3 Machine Image

Machine Image 用于保存 Compute Engine VM 的整体状态和配置。

通常包含：

- VM 配置。
    
- 多个附加磁盘的数据。
    
- VM Metadata。
    
- 部分相关配置和权限信息。
    

适用场景：

- VM 克隆。
    
- VM 备份和恢复。
    
- 迁移或复制 VM 配置。
    

Machine Image 比单个 Disk Snapshot 保存的信息更完整。

### 4.4 Instance Template

Instance Template 保存创建 VM 所需的配置模板，例如：

- Machine Type。
    
- Boot Disk 配置。
    
- Network Interface。
    
- Service Account。
    
- Metadata。
    
- Tags。
    

它主要用于创建 VM，而不是备份已有 VM 的磁盘数据。

常见场景：

- Managed Instance Group（MIG）。
    
- 自动扩缩容。
    
- 批量创建相同配置的 VM。
    

### 4.5 四者对比

|资源|保存磁盘数据|保存 VM 配置|主要用途|
|---|---|---|---|
|Disk Snapshot|是，单个磁盘|否|磁盘备份和恢复|
|Custom Image|是，镜像内容|不完整|创建标准化磁盘或 VM|
|Machine Image|是，可包含多个磁盘|是，包含较完整的实例配置|VM 备份、克隆、恢复|
|Instance Template|否|是，创建 VM 所需配置|批量部署 VM|

### 4.6 Disk → Snapshot → Image

可以通过以下流程制作自定义镜像：

```
Existing Boot Disk
       |
       v
   Snapshot
       |
       v
   Custom Image
       |
       v
Create New Boot Disk
       |
       v
   New VM
```

这种方式可以先获取磁盘某个时间点的 Snapshot，再从 Snapshot 创建 Image，避免直接以持续写入的原始磁盘作为镜像来源。

但需要注意：

- Snapshot 不会自动保证应用程序级别的数据一致性。
    
- 对数据库等正在写入的系统，应该先执行适当的应用一致性操作，例如 Flush、暂停写入或使用数据库原生备份机制。
    
- 直接从 Disk 创建 Image 也是支持的，并不是必须先创建 Snapshot。
    
- Snapshot 和 Image 都不能替代完整的灾难恢复设计。
    

### 4.7 如何选择？

|需求|推荐资源|
|---|---|
|备份一个数据盘|Snapshot|
|恢复一个磁盘|Snapshot|
|创建包含预装软件的标准 Boot Image|Custom Image|
|备份整个 VM，包括多个磁盘和配置|Machine Image|
|批量创建相同配置的 VM|Instance Template|
|创建自动扩缩容 VM 集群|Instance Template + MIG|

面试问题：

1. Disk Snapshot、Custom Image 和 Machine Image 有什么区别？
    
2. 如果需要备份一台包含 Boot Disk 和 Data Disk 的 VM，应该使用什么？
    
3. Instance Template 是否包含 VM 的实际磁盘数据？
    
4. 如何使用 Snapshot 创建新的 VM？
    
5. 为什么数据库备份不能只考虑磁盘 Snapshot？
    
6. MIG 如何利用 Instance Template 批量创建 VM？
    

## 5. Sole-Tenant Nodes

### 5.1 核心概念

Sole-Tenant Node 是专用物理服务器资源，允许用户的 VM 独占指定宿主机，避免与其他客户的 VM 共享物理服务器。

适用场景：

- 数据隔离要求。
    
- 合规要求。
    
- License 绑定物理 CPU 或 Core 的场景。
    
- 需要控制 VM 物理部署位置的场景。
    

注意：Sole-Tenant 不代表一台物理服务器只能运行一台 VM。同一台 Sole-Tenant Node 上可以运行多个属于同一租户或符合共享规则的 VM。

### 5.2 Node Template、Node Group、Node Affinity

Sole-Tenant 的主要资源结构：

```
Node Template
     |
     v
Node Group
     |
     v
Sole-Tenant Nodes
     |
     v
VM Instances
```

Node Template

定义 Node 的配置，例如：

- Node Type。
    
- CPU 和 Memory 规格。
    
- Custom Node Affinity Labels。
    

Node Group

一组具有相同配置的 Sole-Tenant Nodes，位于同一个 Zone。

可以：

- 手动调整 Node 数量。
    
- 配置 Autoscaling。
    
- 管理 Host Maintenance Policy。
    
- 管理 Node Sharing。
    

Node Affinity

通过 Affinity Labels 控制 VM 应该部署在哪些 Node 上。

例如：

```
VM A
  |
  v
Node Affinity: workload=finance
  |
  v
Sole-Tenant Node with workload=finance
```

Node Anti-Affinity

用于避免某些 VM 被调度到同一个 Node。

例如，将两个关键业务 VM 放在不同的物理 Node 上，降低单台物理服务器故障带来的影响。

### 5.3 Sole-Tenant Autoscaling

Sole-Tenant Node Group 支持 Autoscaling。

自动扩容：

- 当现有 Node 没有足够容量部署新 VM 时，增加 Node 数量。

自动缩容：

- 当存在空闲 Node 时，可以减少 Node 数量，降低成本。

注意：这里的 Autoscaling 是调整 Sole-Tenant Node Group 的物理 Node 数量，不等同于 MIG 根据 CPU 使用率增加或删除 VM。

面试问题：

1. Sole-Tenant Node 解决什么问题？
    
2. Node Template、Node Group 和 Node Affinity 有什么区别？
    
3. Sole-Tenant Node 是否只能运行一台 VM？
    
4. Node Affinity 和 Anti-Affinity 有什么区别？
    
5. Sole-Tenant Node Group 如何自动扩缩容？
    
6. 如果企业有 BYOL（Bring Your Own License）要求，为什么可以考虑 Sole-Tenant Nodes？
    

## 6. Managed Instance Group（MIG）与 Autoscaling

### 6.1 Managed Instance Group

MIG 是一组由 Google Cloud 自动管理的相同 VM 实例。

主要能力：

- 自动创建和删除 VM。
    
- 自动修复异常 VM（Autohealing）。
    
- 自动扩缩容。
    
- 滚动更新。
    
- 与 Load Balancer 集成。
    

常见场景：

- Web Application。
    
- API Backend。
    
- 无状态计算任务。
    
- 需要水平扩展的服务。
    

### 6.2 Autoscaling Policy

MIG 可以根据不同的指标自动增加或减少 VM 数量。

常见 Autoscaling Signals：

|Signal|说明|
|---|---|
|CPU Utilization|根据平均 CPU 使用率扩缩容|
|Load Balancing Serving Capacity|根据负载均衡后端的服务容量扩缩容|
|Cloud Monitoring Metrics|根据 Cloud Monitoring 指标扩缩容|
|Schedule|根据预先定义的时间计划调整实例数量|

例如：

```
Average CPU > Target
          |
          v
      Scale Out
          |
          v
    Add VM Instances
```

当负载下降时：

```
Average CPU < Target
          |
          v
      Scale In
          |
          v
   Remove VM Instances
```

### 6.3 Target Utilization

例如设置：

```
Target CPU Utilization = 60%
```

Autoscaler 会根据 MIG 的平均 CPU 使用率调整实例数量，使系统尽可能接近目标利用率。

注意：

- Target Utilization 是扩缩容目标，不是硬性上限。
    
- 扩缩容存在时间延迟。
    
- VM 启动时间和负载变化速度会影响扩缩容效果。
    
- 可以设置 Minimum 和 Maximum Replicas，限制实例数量范围。
    

### 6.4 Cloud Monitoring Metrics Autoscaling

除了 CPU，也可以根据自定义或内置 Monitoring Metrics 扩缩容。

例如：

- Pub/Sub backlog。
    
- 请求延迟。
    
- 活跃用户数量。
    
- 网络流量。
    
- 应用程序处理任务数量。
    

有两种常见方式：

Utilization Target

根据指标目标值进行扩缩容。

例如：

```
Target Memory Usage = 70%
```

Single Instance Assignment

根据总工作量和每个 VM 能处理的工作量计算所需实例数量。

例如：

```
Total Pending Tasks = 1000
Tasks per VM = 100

Required VMs = 1000 / 100 = 10
```

### 6.5 Autoscaling 的重要限制

- MIG Autoscaler 不会因为任何单个 VM CPU 高就一定扩容，而是根据配置的指标及其聚合方式判断。
    
- 设置多个 Scaling Signals 时，Autoscaler 会分别计算需要的实例数量，并选择其中最大的建议规模。
    
- Autoscaling 需要配置 Minimum 和 Maximum Replicas 等边界。
    
- Scale In 可能受 Stabilization Period 等机制影响，避免频繁删除 VM。
    

面试问题：

1. MIG 的作用是什么？与 Unmanaged Instance Group 有什么区别？
    
2. MIG 支持哪些 Autoscaling Signals？
    
3. 如何根据 Pub/Sub backlog 自动扩容 Data Processing VM？
    
4. Target Utilization 和 Single Instance Assignment 有什么区别？
    
5. 如果同时配置 CPU 和 Monitoring Metrics，Autoscaler 如何决定实例数量？
    
6. MIG 的 Autohealing 和 Autoscaling 有什么区别？
    

## 7. Cloud Monitoring：Metric Kind 与 Memory Metrics

### 7.1 Metric Kind

Google Cloud Monitoring 中，Metric Kind 描述指标数据点的时间语义。

主要有三种：

GAUGE

表示某个时间点的当前值。

例如：

- 当前 CPU 使用率。
    
- 当前 Memory 使用量。
    
- 当前磁盘使用量。
    
- 当前队列长度。
    

```
Time:   T1   T2   T3
Value:  30   45   60
```

每个数据点代表采集时刻的状态。

DELTA

表示一个时间区间内发生的变化量。

例如：

- 一分钟内收到的请求数量。
    
- 一分钟内处理的消息数量。
    
- 一分钟内发生的错误次数。
    

```
Interval 1: 100 requests
Interval 2: 150 requests
Interval 3: 120 requests
```

每个值代表对应时间区间内的增量。

CUMULATIVE

表示从某个起始时间开始不断累计的值。

例如：

- VM 启动以来发送的总字节数。
    
- 进程启动以来累计的请求数量。
    

```
Time:   T1    T2    T3
Value:  100   250   400
```

### 7.2 对比

|Metric Kind|含义|典型例子|
|---|---|---|
|GAUGE|某个时刻的状态值|CPU、Memory|
|DELTA|一个时间窗口内的变化量|每分钟请求数|
|CUMULATIVE|从起始时间持续累计的值|总发送字节数|

注意：

CUMULATIVE 指标通常需要通过 Rate、Delta 等转换或聚合方式，才能更方便地用于图表和告警。

### 7.3 Memory Usage

Linux 内存管理中，不能简单地把 `used` 理解为应用程序实际占用的全部内存，也不能直接把 `free` 理解为系统可用内存。

Linux 会利用空闲内存作为：

- Buffer：用于块设备 I/O 等缓存。
    
- Page Cache：缓存文件内容。
    
- Slab：内核用于管理对象的缓存。
    

这些内存可以在需要时被回收或重新使用。

常见 Linux 命令：

```
free -h
```

其中：

- `total`：总内存。
    
- `used`：已使用内存，具体计算方式取决于系统版本和工具。
    
- `free`：完全空闲内存。
    
- `buff/cache`：用于 Buffer 和 Cache 等用途的内存。
    
- `available`：系统估计可供新应用使用的内存。
    

因此，监控 Linux 内存时，通常应该关注 `MemAvailable` 或对应的 Available Memory 指标，而不是只看 `free`。

### 7.4 Memory Metrics 与 Autoscaling

如果使用 Memory Usage 作为 MIG Autoscaling 的指标，需要先明确指标的定义：

- 是 Used Memory？
    
- 是 Available Memory？
    
- 是 Memory Utilization Percentage？
    
- 是否包含 Cache 和 Buffer？
    
- 指标是 GAUGE 还是 DELTA？
    

不能简单地认为所有 Monitoring Memory 指标默认都不包含 Buffer、Cache 和 Slab，再统一用 `used + buffered + cached + slab` 计算。

不同 Agent 和指标定义可能采用不同的计算方式。

如果目标是评估 Linux 系统是否有内存压力，应该优先选择能反映可用内存或内存压力的指标。

面试问题：

1. GAUGE、DELTA 和 CUMULATIVE 有什么区别？
    
2. 如果要监控当前 VM Memory Usage，应该使用哪种 Metric Kind？
    
3. 如果要统计每分钟处理的消息数量，应该使用哪种 Metric Kind？
    
4. 为什么 Linux 的 Free Memory 不等于 Available Memory？
    
5. Linux Page Cache 和 Buffer 是否代表内存浪费？
    
6. 如何利用 Cloud Monitoring 的自定义指标进行 MIG Autoscaling？
    

## 8. VM Manager、OS Config Agent 与 Ops Agent

### 8.1 VM Manager

VM Manager 是 Google Cloud 提供的 VM 操作系统管理能力，基于 OS Config 服务和 OS Config Agent。

主要功能：

- OS Patch Management。
    
- OS Inventory Management。
    
- OS Policy Management。
    
- 批量管理 VM 操作系统配置。
    

### 8.2 OS Config Agent

OS Config Agent 运行在 VM 内部，负责与 Google Cloud OS Config 服务通信，并执行管理任务。

例如：

- 检查操作系统补丁。
    
- 下载并安装系统补丁。
    
- 执行 OS Policy。
    
- 收集操作系统 Inventory。
    

### 8.3 Patch Management

VM Manager 可以对多个 VM 执行批量 Patch Job。

典型流程：

```
Cloud VM Manager
       |
       v
   Patch Job
       |
       v
   OS Config API
       |
       v
   OS Config Agent
       |
       v
Install OS Updates
```

适用场景：

- 定期更新 Linux 系统补丁。
    
- 更新 Windows 系统补丁。
    
- 管理大量 Compute Engine VM。
    
- 统一管理操作系统安全更新。
    

### 8.4 Ops Agent

Ops Agent 是安装在 VM 内部的监控 Agent，主要用于收集：

- Logs。
    
- Metrics。
    

并发送到：

- Cloud Logging。
    
- Cloud Monitoring。
    

常见用途：

- 收集应用日志。
    
- 收集系统日志。
    
- 收集 CPU、Memory、Disk、Network 等指标。
    
- 监控 VM 运行状态。
    

### 8.5 OS Config Agent vs Ops Agent

|对比|OS Config Agent|Ops Agent|
|---|---|---|
|主要用途|操作系统配置和管理|日志和指标采集|
|典型功能|Patch、Inventory、OS Policy|Logs、Metrics|
|对应服务|VM Manager / OS Config|Cloud Logging / Monitoring|
|主要目标|管理和维护 VM|观察和监控 VM|

面试问题：

1. VM Manager 解决什么问题？
    
2. OS Config Agent 的作用是什么？
    
3. 如何批量给上百台 Compute Engine VM 安装安全补丁？
    
4. Ops Agent 和 OS Config Agent 有什么区别？
    
5. 如果需要收集 VM 上的 Application Logs 和 System Metrics，应该使用什么？
    
6. Patch Job 失败时，应该检查哪些方面？
    

## 9. Compute Engine 核心概念总结

|模块|核心知识点|主要用途|
|---|---|---|
|Persistent Disk|pd-standard、pd-balanced、pd-ssd、pd-extreme、Hyperdisk|持久化块存储|
|Local SSD|高性能、临时性|Cache、临时计算|
|Disk Expansion|`resize2fs`、`xfs_growfs`|在线扩容文件系统|
|Cloud Shell|5 GB `$HOME` 持久化存储|云端开发环境|
|App Engine|Standard、Flexible|PaaS 应用部署|
|Disk Snapshot|时间点磁盘备份|磁盘恢复|
|Custom Image|标准化磁盘镜像|创建相同系统环境|
|Machine Image|VM 配置和多磁盘数据|VM 备份和克隆|
|Instance Template|VM 配置模板|批量创建 VM|
|Sole-Tenant|Node Template、Node Group、Affinity|专用物理主机|
|MIG|Autohealing、Autoscaling、Rolling Update|VM 水平扩展|
|Cloud Monitoring|GAUGE、DELTA、CUMULATIVE|指标采集与分析|
|VM Manager|OS Config Agent、Patch Job|VM 操作系统管理|
|Ops Agent|Logs、Metrics|VM 监控和日志采集|