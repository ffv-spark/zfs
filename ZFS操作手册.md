# ZFS 文件系统操作手册

## 目录

1. [ZFS 简介](#1-zfs-简介)
2. [ZFS 核心概念](#2-zfs-核心概念)
3. [存储池管理（zpool）](#3-存储池管理zpool)
4. [数据集管理（zfs）](#4-数据集管理zfs)
5. [快照与克隆](#5-快照与克隆)
6. [数据发送与接收](#6-数据发送与接收)
7. [属性管理](#7-属性管理)
8. [性能优化](#8-性能优化)
9. [故障排查与恢复](#9-故障排查与恢复)
10. [加密功能](#10-加密功能)
11. [配额与预留](#11-配额与预留)
12. [高级功能](#12-高级功能)
13. [最佳实践](#13-最佳实践)

---

## 1. ZFS 简介

### 1.1 什么是 ZFS

ZFS（Zettabyte File System）是一个集文件系统和卷管理器于一体的高级存储系统，具有以下特点：

- **数据完整性**：端到端校验和，自动检测和修复数据损坏
- **存储池**：简化的存储管理，多个物理设备组成单一存储池
- **快照和克隆**：即时快照，几乎零开销的克隆
- **压缩和去重**：内置数据压缩和去重功能
- **RAID-Z**：软件 RAID 实现，避免写入漏洞
- **ARC 缓存**：自适应替换缓存，智能内存管理
- **数据加密**：原生加密支持

### 1.2 架构层次

```
用户命令层:
  ├── zpool     # 存储池管理
  ├── zfs       # 数据集管理
  ├── zdb       # 调试工具
  └── zed       # 事件守护进程

用户空间库:
  ├── libzfs          # 主要用户空间接口
  ├── libzfs_core     # 核心系统调用接口（稳定 ABI）
  └── libzpool        # 用户空间 DMU/SPA

内核模块:
  ├── ZPL    # ZFS POSIX 层
  ├── DSL    # 数据集和快照层
  ├── DMU    # 数据管理单元
  ├── SPA    # 存储池分配器
  └── VDEV   # 虚拟设备层
```

---

## 2. ZFS 核心概念

### 2.1 存储池（Storage Pool）

存储池是 ZFS 的基础，它是物理存储设备的逻辑集合。一个池可以包含多个虚拟设备（vdev），所有数据集共享池中的空间。

### 2.2 虚拟设备（Virtual Device, vdev）

虚拟设备是存储池的组成单元，支持以下类型：

- **disk**：单个磁盘或分区
- **mirror**：镜像，数据复制到多个设备
- **raidz1/raidz2/raidz3**：分布式奇偶校验 RAID（单/双/三重校验）
- **draid1/draid2/draid3**：分布式 RAID，支持集成热备盘
- **spare**：热备盘
- **log**：独立的意图日志设备（SLOG）
- **cache**：L2ARC 缓存设备
- **special**：特殊分配类（元数据和小文件）
- **dedup**：去重表专用设备

### 2.3 数据集（Dataset）

数据集是 ZFS 中的核心存储单元，包括：

- **文件系统（filesystem）**：可挂载的标准文件系统
- **卷（volume）**：块设备，也称为 zvol
- **快照（snapshot）**：某个时间点的只读副本
- **书签（bookmark）**：轻量级快照标记

### 2.4 快照（Snapshot）

快照是数据集在某个时间点的只读副本：
- 创建速度极快
- 初始不占用额外空间
- 通过写时复制（COW）实现
- 命名格式：`数据集名@快照名`

### 2.5 克隆（Clone）

克隆是从快照创建的可写数据集：
- 初始与快照内容相同
- 独立修改不影响原快照
- 共享存储空间直到修改发生

### 2.6 属性（Properties）

ZFS 使用属性来控制行为和显示统计信息：
- **原生属性**：由 ZFS 定义，控制行为
- **用户属性**：用户自定义，用于标注

---

## 3. 存储池管理（zpool）

### 3.1 创建存储池

#### 3.1.1 基本语法

```bash
zpool create [选项] 池名 vdev配置 [vdev...]
```

#### 3.1.2 单盘池（不推荐用于生产）

```bash
# 使用单个磁盘创建池
zpool create mypool /dev/sda

# 使用文件创建池（仅用于测试）
zpool create testpool /path/to/file
```

#### 3.1.3 镜像池

```bash
# 创建双盘镜像
zpool create mypool mirror /dev/sda /dev/sdb

# 创建多个镜像组
zpool create mypool \
    mirror /dev/sda /dev/sdb \
    mirror /dev/sdc /dev/sdd
```

#### 3.1.4 RAID-Z 池

```bash
# RAID-Z1（单奇偶校验，类似 RAID-5）
zpool create mypool raidz /dev/sda /dev/sdb /dev/sdc

# RAID-Z2（双奇偶校验，类似 RAID-6）
zpool create mypool raidz2 /dev/sda /dev/sdb /dev/sdc /dev/sdd

# RAID-Z3（三重奇偶校验）
zpool create mypool raidz3 /dev/sd{a,b,c,d,e}
```

#### 3.1.5 dRAID 池

```bash
# 创建 dRAID1 池
zpool create mypool draid /dev/sd{a,b,c,d,e,f,g,h}

# 创建带配置的 dRAID2 池
# 格式：draid[奇偶数]:数据盘d:子设备数c:热备数s
zpool create mypool draid2:4d:12c:1s /dev/sd{a..l}
```

#### 3.1.6 创建选项

```bash
# 强制创建（覆盖现有文件系统）
zpool create -f mypool /dev/sda

# 指定挂载点
zpool create -m /mnt/mypool mypool /dev/sda

# 不挂载
zpool create -m none mypool /dev/sda

# 设置 ashift（扇区大小）
# 对于 4K 扇区磁盘使用 ashift=12 (2^12=4096)
zpool create -o ashift=12 mypool /dev/sda

# 创建时启用压缩
zpool create -O compression=lz4 mypool /dev/sda

# 禁用 atime 更新（提高性能）
zpool create -O atime=off mypool /dev/sda
```

#### 3.1.7 复杂池配置

```bash
# 混合配置：数据 + 日志 + 缓存
zpool create mypool \
    raidz2 /dev/sd{a,b,c,d,e,f} \
    log mirror /dev/sdg /dev/sdh \
    cache /dev/sdi /dev/sdj

# 带热备盘的配置
zpool create mypool \
    raidz2 /dev/sd{a,b,c,d,e} \
    spare /dev/sdf /dev/sdg

# 使用特殊设备（存储元数据和小文件）
zpool create mypool \
    raidz2 /dev/sd{a,b,c,d,e,f} \
    special mirror /dev/nvme0n1 /dev/nvme1n1
```

### 3.2 查看池信息

#### 3.2.1 列出所有池

```bash
# 列出所有池
zpool list

# 显示详细信息
zpool list -v

# 显示特定属性
zpool list -o name,size,allocated,free,fragmentation,health

# 脚本友好格式（无标题，制表符分隔）
zpool list -H -o name,size
```

#### 3.2.2 查看池状态

```bash
# 显示池的健康状态
zpool status

# 显示特定池的状态
zpool status mypool

# 显示详细错误信息
zpool status -v

# 显示所有池的状态
zpool status -x
```

#### 3.2.3 查看 I/O 统计

```bash
# 显示 I/O 统计
zpool iostat

# 持续显示（每 2 秒更新一次）
zpool iostat 2

# 显示特定池，10 次采样
zpool iostat mypool 2 10

# 显示详细统计
zpool iostat -v

# 显示延迟直方图
zpool iostat -l
```

### 3.3 管理池设备

#### 3.3.1 添加设备

```bash
# 添加 vdev 到池
zpool add mypool raidz /dev/sd{g,h,i}

# 添加日志设备
zpool add mypool log mirror /dev/sdj /dev/sdk

# 添加缓存设备
zpool add mypool cache /dev/sdl

# 添加热备盘
zpool add mypool spare /dev/sdm
```

#### 3.3.2 移除设备

```bash
# 移除日志设备
zpool remove mypool /dev/sdj

# 移除缓存设备
zpool remove mypool /dev/sdl

# 移除热备盘
zpool remove mypool /dev/sdm

# 移除 vdev（仅支持特定类型）
zpool remove mypool raidz-1

# 取消正在进行的移除操作
zpool remove -s mypool
```

#### 3.3.3 替换设备

```bash
# 替换故障设备
zpool replace mypool /dev/old-disk /dev/new-disk

# 原地替换（扩容）
zpool replace mypool /dev/sda
```

#### 3.3.4 附加和分离设备（镜像）

```bash
# 将设备附加到现有设备（创建镜像）
zpool attach mypool /dev/sda /dev/sdb

# 从镜像中分离设备
zpool detach mypool /dev/sdb
```

#### 3.3.5 上线和下线设备

```bash
# 使设备下线
zpool offline mypool /dev/sda

# 临时下线（重启后自动上线）
zpool offline -t mypool /dev/sda

# 使设备上线
zpool online mypool /dev/sda

# 上线并扩展设备
zpool online -e mypool /dev/sda
```

### 3.4 导入和导出池

#### 3.4.1 导出池

```bash
# 导出池（卸载）
zpool export mypool

# 强制导出
zpool export -f mypool
```

#### 3.4.2 导入池

```bash
# 列出可导入的池
zpool import

# 搜索特定目录
zpool import -d /dev/disk/by-id

# 导入池
zpool import mypool

# 使用不同的名称导入
zpool import oldname newname

# 强制导入（之前在其他系统使用）
zpool import -f mypool

# 只读导入
zpool import -o readonly=on mypool

# 使用备用根目录导入
zpool import -R /mnt mypool

# 恢复已销毁的池
zpool import -D mypool
```

### 3.5 池维护操作

#### 3.5.1 清洗（Scrub）

```bash
# 开始清洗
zpool scrub mypool

# 暂停清洗
zpool scrub -p mypool

# 停止清洗
zpool scrub -s mypool

# 从上次停止的地方继续清洗
zpool scrub -C mypool
```

#### 3.5.2 重新银化（Resilver）

```bash
# 替换设备时自动触发
zpool replace mypool /dev/old /dev/new

# 暂停重新银化
zpool resilver -p mypool

# 恢复重新银化
zpool resilver mypool
```

#### 3.5.3 TRIM 操作

```bash
# 手动 TRIM
zpool trim mypool

# TRIM 特定设备
zpool trim mypool /dev/sda

# 取消 TRIM
zpool trim -c mypool

# 暂停 TRIM
zpool trim -s mypool

# 查看 TRIM 状态
zpool status -t mypool
```

#### 3.5.4 初始化（Initialize）

```bash
# 初始化所有设备
zpool initialize mypool

# 初始化特定设备
zpool initialize mypool /dev/sda

# 取消初始化
zpool initialize -c mypool

# 暂停初始化
zpool initialize -s mypool

# 查看初始化状态
zpool status -i mypool
```

### 3.6 池检查点

```bash
# 创建检查点
zpool checkpoint mypool

# 列出检查点信息
zpool status mypool

# 丢弃检查点
zpool checkpoint -d mypool

# 导入并回滚到检查点
zpool import --rewind-to-checkpoint mypool
```

### 3.7 清除错误

```bash
# 清除设备错误计数
zpool clear mypool

# 清除特定设备的错误
zpool clear mypool /dev/sda
```

### 3.8 池升级

```bash
# 检查池版本
zpool upgrade

# 查看可升级的功能
zpool upgrade -v

# 升级池
zpool upgrade mypool

# 升级所有池
zpool upgrade -a
```

### 3.9 池历史

```bash
# 查看池历史命令
zpool history mypool

# 显示详细时间戳
zpool history -l mypool

# 显示内部事件
zpool history -i mypool
```

### 3.10 池属性管理

```bash
# 获取属性
zpool get all mypool
zpool get size,free,allocated mypool

# 设置属性
zpool set comment="生产数据池" mypool
zpool set autoexpand=on mypool
zpool set autotrim=on mypool

# 列出所有可设置的属性
zpool get all mypool
```

### 3.11 销毁池

```bash
# 销毁池（不可恢复）
zpool destroy mypool

# 强制销毁
zpool destroy -f mypool
```

---

## 4. 数据集管理（zfs）

### 4.1 创建数据集

#### 4.1.1 创建文件系统

```bash
# 创建基本文件系统
zfs create mypool/data

# 创建嵌套文件系统
zfs create mypool/data/documents
zfs create mypool/data/pictures

# 指定挂载点
zfs create -o mountpoint=/srv/data mypool/data

# 创建时设置多个属性
zfs create \
    -o compression=lz4 \
    -o atime=off \
    -o recordsize=1M \
    mypool/data
```

#### 4.1.2 创建卷（zvol）

```bash
# 创建 10GB 卷
zfs create -V 10G mypool/vol1

# 创建稀疏卷
zfs create -s -V 100G mypool/sparse-vol

# 设置块大小
zfs create -V 10G -o volblocksize=16K mypool/vol2
```

### 4.2 列出数据集

```bash
# 列出所有数据集
zfs list

# 列出特定池的数据集
zfs list -r mypool

# 显示快照
zfs list -t snapshot

# 显示所有类型
zfs list -t all

# 自定义列
zfs list -o name,used,available,referenced,mountpoint

# 递归显示
zfs list -r mypool/data

# 按某列排序
zfs list -S used

# 脚本模式
zfs list -H -o name
```

### 4.3 销毁数据集

```bash
# 销毁文件系统
zfs destroy mypool/data

# 递归销毁（包括子数据集和快照）
zfs destroy -r mypool/data

# 递归销毁克隆
zfs destroy -R mypool/data

# 强制卸载并销毁
zfs destroy -f mypool/data

# 执行测试（不实际销毁）
zfs destroy -n -v mypool/data

# 延迟销毁（当不再使用时）
zfs destroy -d mypool/data@snapshot
```

### 4.4 重命名数据集

```bash
# 重命名数据集
zfs rename mypool/old-name mypool/new-name

# 创建父数据集（如果不存在）
zfs rename -p mypool/data mypool/production/data

# 强制卸载并重命名
zfs rename -f mypool/data mypool/newdata
```

### 4.5 挂载和卸载

```bash
# 挂载文件系统
zfs mount mypool/data

# 挂载所有文件系统
zfs mount -a

# 卸载文件系统
zfs umount mypool/data

# 强制卸载
zfs umount -f mypool/data

# 查看已挂载的文件系统
zfs mount
```

---

## 5. 快照与克隆

### 5.1 快照操作

#### 5.1.1 创建快照

```bash
# 创建快照
zfs snapshot mypool/data@snapshot1

# 创建递归快照（包括所有子数据集）
zfs snapshot -r mypool/data@snapshot1

# 创建多个数据集的快照
zfs snapshot mypool/data@snap1 mypool/backup@snap1
```

#### 5.1.2 列出快照

```bash
# 列出所有快照
zfs list -t snapshot

# 列出特定数据集的快照
zfs list -t snapshot -r mypool/data

# 显示快照空间使用
zfs list -t snapshot -o name,used,referenced
```

#### 5.1.3 访问快照

```bash
# 快照位于 .zfs/snapshot 目录
ls /mypool/data/.zfs/snapshot/

# 访问快照内容
cd /mypool/data/.zfs/snapshot/snapshot1/
```

#### 5.1.4 销毁快照

```bash
# 销毁单个快照
zfs destroy mypool/data@snapshot1

# 销毁多个快照
zfs destroy mypool/data@snap1,snap2,snap3

# 销毁范围内的快照
zfs destroy mypool/data@snap1%snap5

# 递归销毁快照
zfs destroy -r mypool/data@snapshot1

# 延迟销毁
zfs destroy -d mypool/data@snapshot1
```

#### 5.1.5 回滚快照

```bash
# 回滚到快照
zfs rollback mypool/data@snapshot1

# 销毁较新的快照并回滚
zfs rollback -r mypool/data@snapshot1

# 强制回滚
zfs rollback -f mypool/data@snapshot1
```

#### 5.1.6 快照保持（Hold）

```bash
# 添加保持标记
zfs hold keep1 mypool/data@snapshot1

# 列出保持标记
zfs holds mypool/data@snapshot1

# 释放保持标记
zfs release keep1 mypool/data@snapshot1

# 递归添加保持
zfs hold -r keep1 mypool/data@snapshot1
```

### 5.2 克隆操作

#### 5.2.1 创建克隆

```bash
# 从快照创建克隆
zfs clone mypool/data@snapshot1 mypool/clone1

# 创建克隆并设置属性
zfs clone -o compression=gzip mypool/data@snap1 mypool/clone1
```

#### 5.2.2 提升克隆

```bash
# 提升克隆（使克隆成为独立数据集）
zfs promote mypool/clone1

# 查看克隆依赖关系
zfs list -t all -o name,origin
```

#### 5.2.3 查看克隆信息

```bash
# 查看克隆的来源
zfs get origin mypool/clone1

# 查看快照的克隆列表
zfs get clones mypool/data@snapshot1
```

### 5.3 书签操作

```bash
# 创建书签
zfs bookmark mypool/data@snapshot1 mypool/data#bookmark1

# 列出书签
zfs list -t bookmark

# 销毁书签
zfs destroy mypool/data#bookmark1
```

### 5.4 快照差异

```bash
# 查看快照之间的差异
zfs diff mypool/data@snap1 mypool/data@snap2

# 查看快照与当前状态的差异
zfs diff mypool/data@snap1

# 显示文件类型
zfs diff -F mypool/data@snap1

# 显示时间戳
zfs diff -t mypool/data@snap1
```

---

## 6. 数据发送与接收

### 6.1 基本发送和接收

#### 6.1.1 完整发送

```bash
# 发送快照到文件
zfs send mypool/data@snapshot1 > /backup/data.zfs

# 通过 SSH 发送到远程系统
zfs send mypool/data@snapshot1 | ssh user@remote "zfs receive backuppool/data"

# 使用压缩传输
zfs send mypool/data@snapshot1 | gzip | ssh user@remote "gunzip | zfs receive backuppool/data"
```

#### 6.1.2 增量发送

```bash
# 发送增量快照
zfs send -i mypool/data@snap1 mypool/data@snap2 > /backup/incremental.zfs

# 增量发送到远程
zfs send -i mypool/data@snap1 mypool/data@snap2 | \
    ssh user@remote "zfs receive backuppool/data"
```

### 6.2 高级发送选项

#### 6.2.1 递归发送

```bash
# 递归发送所有子数据集
zfs send -R mypool/data@snapshot1 | \
    ssh user@remote "zfs receive -F backuppool/data"
```

#### 6.2.2 复制发送

```bash
# 复制模式（包括所有快照）
zfs send -R mypool/data@latest | \
    ssh user@remote "zfs receive -F backuppool/data"
```

#### 6.2.3 中间快照发送

```bash
# 使用中间快照（-I）
zfs send -I mypool/data@base mypool/data@latest | \
    ssh user@remote "zfs receive backuppool/data"
```

#### 6.2.4 从书签发送

```bash
# 从书签进行增量发送
zfs send -i mypool/data#bookmark1 mypool/data@snap2 > /backup/inc.zfs
```

### 6.3 高级接收选项

#### 6.3.1 强制回滚接收

```bash
# 强制回滚接收端
zfs receive -F backuppool/data < /backup/data.zfs
```

#### 6.3.2 恢复中断的接收

```bash
# 可恢复接收
zfs receive -s backuppool/data < /backup/data.zfs

# 获取恢复令牌
zfs get receive_resume_token backuppool/data

# 使用令牌恢复发送
zfs send -t <token> | zfs receive backuppool/data
```

#### 6.3.3 其他接收选项

```bash
# 不挂载接收的文件系统
zfs receive -u backuppool/data < /backup/data.zfs

# 详细模式
zfs receive -v backuppool/data < /backup/data.zfs

# 测试模式
zfs receive -n -v backuppool/data < /backup/data.zfs
```

### 6.4 原始发送（加密）

```bash
# 原始发送（用于加密数据集）
zfs send -w mypool/encrypted@snap1 | \
    ssh user@remote "zfs receive backuppool/encrypted"

# 原始递归发送
zfs send -Rw mypool/encrypted@snap1 | \
    ssh user@remote "zfs receive -F backuppool/encrypted"
```

### 6.5 编辑发送流

```bash
# 重写发送流
zfs send mypool/data@snap1 | \
    zstream rewrite -o feature1=disabled | \
    zfs receive newpool/data
```

### 6.6 实用脚本示例

#### 6.6.1 自动备份脚本

```bash
#!/bin/bash
# 增量备份脚本

POOL="mypool"
DATASET="data"
REMOTE_HOST="backup-server"
REMOTE_POOL="backuppool"
SNAPSHOT_PREFIX="auto"

# 创建新快照
SNAPSHOT="${SNAPSHOT_PREFIX}-$(date +%Y%m%d-%H%M%S)"
zfs snapshot ${POOL}/${DATASET}@${SNAPSHOT}

# 查找上一个快照
LAST_SNAPSHOT=$(zfs list -H -t snapshot -o name -S creation ${POOL}/${DATASET} | \
    grep "@${SNAPSHOT_PREFIX}" | head -2 | tail -1)

# 增量发送
if [ -n "$LAST_SNAPSHOT" ]; then
    zfs send -i ${LAST_SNAPSHOT} ${POOL}/${DATASET}@${SNAPSHOT} | \
        ssh ${REMOTE_HOST} "zfs receive -F ${REMOTE_POOL}/${DATASET}"
else
    zfs send -R ${POOL}/${DATASET}@${SNAPSHOT} | \
        ssh ${REMOTE_HOST} "zfs receive -F ${REMOTE_POOL}/${DATASET}"
fi
```

---

## 7. 属性管理

### 7.1 查看属性

```bash
# 查看所有属性
zfs get all mypool/data

# 查看特定属性
zfs get compression mypool/data

# 查看多个属性
zfs get compression,checksum,atime mypool/data

# 递归查看
zfs get -r compression mypool

# 显示属性来源
zfs get -s local,inherited compression mypool/data

# 脚本模式
zfs get -H -o name,property,value compression mypool/data
```

### 7.2 设置属性

```bash
# 设置单个属性
zfs set compression=lz4 mypool/data

# 设置多个属性
zfs set compression=lz4 mypool/data
zfs set atime=off mypool/data

# 递归设置
zfs set compression=lz4 -r mypool/data
```

### 7.3 继承属性

```bash
# 继承父数据集的属性
zfs inherit compression mypool/data/child

# 递归继承
zfs inherit -r compression mypool/data

# 恢复到接收的值
zfs inherit -S compression mypool/data
```

### 7.4 重要属性详解

#### 7.4.1 压缩（Compression）

```bash
# 可选值：on, off, lz4, gzip, gzip-[1-9], zstd, zstd-[1-19], zle, lzjb
zfs set compression=lz4 mypool/data          # 推荐，性能好
zfs set compression=zstd mypool/data         # 高压缩比
zfs set compression=gzip-9 mypool/data       # 最高压缩比，慢

# 查看压缩比
zfs get compressratio mypool/data
```

#### 7.4.2 校验和（Checksum）

```bash
# 可选值：on, off, fletcher2, fletcher4, sha256, sha512, skein, edonr
zfs set checksum=sha256 mypool/data
zfs set checksum=edonr mypool/data           # 最快
```

#### 7.4.3 去重（Dedup）

```bash
# 警告：去重需要大量内存！
zfs set dedup=on mypool/data

# 查看去重比
zfs get dedupratio mypool/data

# 建议使用 verify 选项
zfs set dedup=verify mypool/data
```

#### 7.4.4 记录大小（Recordsize）

```bash
# 默认 128K，范围 512B-16M
zfs set recordsize=1M mypool/database        # 大文件/数据库
zfs set recordsize=16K mypool/web            # 小文件
```

#### 7.4.5 配额和预留

```bash
# 设置配额（限制空间使用）
zfs set quota=100G mypool/data

# 设置预留（保证可用空间）
zfs set reservation=50G mypool/data

# 引用配额（不包括子数据集）
zfs set refquota=100G mypool/data

# 引用预留
zfs set refreservation=50G mypool/data
```

#### 7.4.6 挂载点

```bash
# 设置挂载点
zfs set mountpoint=/data mypool/data

# 不挂载
zfs set mountpoint=none mypool/data

# 传统挂载（不由 ZFS 管理）
zfs set mountpoint=legacy mypool/data
```

#### 7.4.7 atime

```bash
# 禁用 atime（提高性能）
zfs set atime=off mypool/data

# 相对 atime
zfs set atime=on mypool/data
zfs set relatime=on mypool/data
```

#### 7.4.8 副本数

```bash
# 设置数据副本数（1-3）
zfs set copies=2 mypool/critical-data
```

#### 7.4.9 同步写入

```bash
# 可选值：standard, always, disabled
zfs set sync=standard mypool/data            # 默认
zfs set sync=disabled mypool/temp            # 危险，但快
zfs set sync=always mypool/database          # 最安全，慢
```

#### 7.4.10 快照可见性

```bash
# 控制 .zfs 目录可见性
zfs set snapdir=visible mypool/data
zfs set snapdir=hidden mypool/data
```

### 7.5 用户属性

```bash
# 设置用户属性（必须包含冒号）
zfs set com.example:department=engineering mypool/data
zfs set com.example:owner=john mypool/data

# 查看用户属性
zfs get com.example:department mypool/data

# 删除用户属性
zfs inherit com.example:department mypool/data
```

---

## 8. 性能优化

### 8.1 ARC（自适应替换缓存）

#### 8.1.1 查看 ARC 统计

```bash
# 查看 ARC 使用情况
cat /proc/spl/kstat/zfs/arcstats

# 使用 arc_summary（更友好）
arc_summary
```

#### 8.1.2 调整 ARC 大小

```bash
# 设置 ARC 最大值（在 /etc/modprobe.d/zfs.conf）
options zfs zfs_arc_max=8589934592  # 8GB

# 设置 ARC 最小值
options zfs zfs_arc_min=1073741824  # 1GB

# 运行时调整（临时）
echo 8589934592 > /sys/module/zfs/parameters/zfs_arc_max
```

### 8.2 L2ARC（二级缓存）

```bash
# 添加 L2ARC 设备
zpool add mypool cache /dev/ssd1

# 移除 L2ARC 设备
zpool remove mypool /dev/ssd1

# 查看 L2ARC 统计
cat /proc/spl/kstat/zfs/arcstats | grep l2
```

### 8.3 ZIL（ZFS 意图日志）

#### 8.3.1 添加 SLOG

```bash
# 添加独立日志设备（推荐镜像）
zpool add mypool log mirror /dev/nvme0n1 /dev/nvme1n1

# 移除 SLOG
zpool remove mypool /dev/nvme0n1
```

#### 8.3.2 调整同步写入

```bash
# 禁用同步写入（危险）
zfs set sync=disabled mypool/temp

# 对于数据库，保持默认或 always
zfs set sync=always mypool/database
```

### 8.4 记录大小优化

```bash
# 针对大文件（视频、数据库）
zfs set recordsize=1M mypool/media

# 针对小文件
zfs set recordsize=16K mypool/mail

# PostgreSQL
zfs set recordsize=8K mypool/postgres

# MySQL/MariaDB
zfs set recordsize=16K mypool/mysql
```

### 8.5 压缩优化

```bash
# LZ4：最佳平衡（推荐）
zfs set compression=lz4 mypool/data

# ZSTD：更高压缩比
zfs set compression=zstd mypool/archive

# GZIP：最高压缩比，CPU 密集
zfs set compression=gzip-9 mypool/cold-storage
```

### 8.6 预取优化

```bash
# 调整预取（/etc/modprobe.d/zfs.conf）
options zfs zfs_prefetch_disable=0           # 启用预取（默认）
options zfs zfs_prefetch_disable=1           # 禁用预取
```

### 8.7 事务组超时

```bash
# 调整 TXG 超时（秒）
echo 10 > /sys/module/zfs/parameters/zfs_txg_timeout
```

### 8.8 特殊分配类

```bash
# 添加特殊设备（SSD）用于元数据
zpool add mypool special mirror /dev/nvme0n1 /dev/nvme1n1

# 设置小文件阈值
zfs set special_small_blocks=32K mypool/data
```

### 8.9 数据库优化建议

#### PostgreSQL

```bash
zfs create -o recordsize=8K \
    -o primarycache=metadata \
    -o logbias=throughput \
    -o atime=off \
    mypool/postgres
```

#### MySQL/MariaDB

```bash
zfs create -o recordsize=16K \
    -o primarycache=metadata \
    -o logbias=throughput \
    -o atime=off \
    mypool/mysql
```

---

## 9. 故障排查与恢复

### 9.1 检查池健康状态

```bash
# 查看池状态
zpool status

# 查看详细错误
zpool status -v

# 仅显示有问题的池
zpool status -x
```

### 9.2 常见设备状态

- **ONLINE**：设备正常工作
- **DEGRADED**：设备有问题但数据可访问
- **FAULTED**：设备故障
- **OFFLINE**：设备被手动下线
- **UNAVAIL**：设备不可用
- **REMOVED**：设备被物理移除

### 9.3 处理设备故障

#### 9.3.1 替换故障设备

```bash
# 使设备下线
zpool offline mypool /dev/sda

# 替换设备
zpool replace mypool /dev/sda /dev/sdb

# 等待 resilver 完成
zpool status mypool
```

#### 9.3.2 清除错误

```bash
# 清除错误计数
zpool clear mypool

# 清除特定设备的错误
zpool clear mypool /dev/sda
```

### 9.4 数据清洗（Scrub）

```bash
# 开始清洗
zpool scrub mypool

# 查看清洗进度
zpool status mypool

# 停止清洗
zpool scrub -s mypool

# 暂停清洗
zpool scrub -p mypool
```

### 9.5 使用 zdb 调试

```bash
# 检查池配置
zdb mypool

# 显示详细信息
zdb -C mypool

# 检查数据集
zdb mypool/data

# 显示所有块指针
zdb -bbbb mypool

# 检查特定对象
zdb -dddd mypool/data 0
```

### 9.6 恢复删除的文件

```bash
# 从快照恢复
cd /mypool/data/.zfs/snapshot/snapshot1
cp deleted-file.txt /mypool/data/

# 回滚到快照
zfs rollback mypool/data@snapshot1
```

### 9.7 导入损坏的池

```bash
# 强制导入
zpool import -f mypool

# 只读导入
zpool import -o readonly=on mypool

# 忽略丢失的日志设备
zpool import -m mypool

# 回滚到检查点
zpool import --rewind-to-checkpoint mypool

# 回滚到特定事务组
zpool import -F -T <txg> mypool
```

### 9.8 池拆分和恢复

```bash
# 从镜像创建新池
zpool split mypool newpool

# 列出可导入的已销毁池
zpool import -D

# 恢复已销毁的池
zpool import -D mypool
```

### 9.9 事件日志

```bash
# 查看 ZFS 事件
zpool events

# 实时监控事件
zpool events -f

# 清除事件日志
zpool events -c
```

### 9.10 内核模块调优

#### 查看可调参数

```bash
# 列出所有 ZFS 参数
cat /sys/module/zfs/parameters/*

# 查看特定参数
cat /sys/module/zfs/parameters/zfs_arc_max
```

#### 运行时调整

```bash
# 增加脏数据限制
echo 4294967296 > /sys/module/zfs/parameters/zfs_dirty_data_max

# 调整扫描速度
echo 100000000 > /sys/module/zfs/parameters/zfs_scan_vdev_limit
```

---

## 10. 加密功能

### 10.1 创建加密数据集

#### 10.1.1 使用密码短语

```bash
# 创建加密文件系统
zfs create -o encryption=on -o keyformat=passphrase mypool/encrypted

# 创建加密卷
zfs create -V 10G -o encryption=on -o keyformat=passphrase mypool/encrypted-vol
```

#### 10.1.2 使用密钥文件

```bash
# 生成密钥文件
dd if=/dev/random of=/root/zfs-key bs=32 count=1
chmod 600 /root/zfs-key

# 创建加密数据集
zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs-key mypool/encrypted
```

#### 10.1.3 加密选项

```bash
# 指定加密算法
zfs create -o encryption=aes-256-gcm -o keyformat=passphrase mypool/encrypted

# 可用算法：
# - aes-128-ccm
# - aes-192-ccm
# - aes-256-ccm
# - aes-128-gcm (默认)
# - aes-192-gcm
# - aes-256-gcm
```

### 10.2 密钥管理

#### 10.2.1 加载密钥

```bash
# 加载密钥（交互式）
zfs load-key mypool/encrypted

# 从文件加载
zfs load-key -L file:///root/zfs-key mypool/encrypted

# 递归加载
zfs load-key -r mypool/encrypted

# 不挂载
zfs load-key -n mypool/encrypted
```

#### 10.2.2 卸载密钥

```bash
# 卸载密钥
zfs unload-key mypool/encrypted

# 递归卸载
zfs unload-key -r mypool/encrypted
```

#### 10.2.3 更改密钥

```bash
# 更改加密密钥
zfs change-key mypool/encrypted

# 更改为密钥文件
zfs change-key -o keyformat=raw -o keylocation=file:///root/new-key mypool/encrypted

# 继承父数据集的密钥
zfs change-key -i mypool/encrypted/child
```

### 10.3 查看加密状态

```bash
# 查看加密属性
zfs get encryption,keyformat,keystatus,keylocation mypool/encrypted

# 查看加密根
zfs get encryptionroot mypool/encrypted
```

### 10.4 原始发送加密数据

```bash
# 发送加密数据（不解密）
zfs send -w mypool/encrypted@snap1 > /backup/encrypted.zfs

# 接收加密数据
zfs receive mypool/encrypted-copy < /backup/encrypted.zfs

# 通过网络传输
zfs send -w mypool/encrypted@snap1 | \
    ssh user@remote "zfs receive backuppool/encrypted"
```

---

## 11. 配额与预留

### 11.1 配额（Quota）

#### 11.1.1 数据集配额

```bash
# 设置配额
zfs set quota=100G mypool/data

# 查看配额
zfs get quota mypool/data

# 取消配额
zfs set quota=none mypool/data
```

#### 11.1.2 引用配额

```bash
# 设置引用配额（不包括子数据集）
zfs set refquota=50G mypool/data

# 查看
zfs get refquota,quota mypool/data
```

#### 11.1.3 用户配额

```bash
# 设置用户配额
zfs set userquota@john=10G mypool/data

# 查看用户配额
zfs get userquota@john mypool/data

# 列出所有用户配额
zfs userspace mypool/data

# 查看用户空间使用
zfs userspace -o name,used,quota,objused,objquota mypool/data
```

#### 11.1.4 组配额

```bash
# 设置组配额
zfs set groupquota@developers=100G mypool/data

# 查看组空间使用
zfs groupspace mypool/data
```

#### 11.1.5 项目配额

```bash
# 设置项目配额
zfs set projectquota@1000=50G mypool/data

# 查看项目空间使用
zfs projectspace mypool/data

# 管理项目 ID
zfs project -s -p 1000 /mypool/data/project1
```

### 11.2 预留（Reservation）

#### 11.2.1 数据集预留

```bash
# 设置预留
zfs set reservation=50G mypool/data

# 查看预留
zfs get reservation mypool/data

# 取消预留
zfs set reservation=none mypool/data
```

#### 11.2.2 引用预留

```bash
# 设置引用预留
zfs set refreservation=30G mypool/data

# 查看
zfs get refreservation,reservation mypool/data
```

### 11.3 空间使用分析

```bash
# 查看空间使用详情
zfs list -o space

# 查看特定数据集的空间分解
zfs list -o name,used,usedsnap,usedds,usedrefreserv,usedchild mypool/data

# 查看可用空间
zfs get available mypool/data
```

---

## 12. 高级功能

### 12.1 委托管理

#### 12.1.1 授予权限

```bash
# 授予创建权限
zfs allow john create mypool/data

# 授予多个权限
zfs allow john create,destroy,mount,snapshot mypool/data

# 授予给组
zfs allow -g developers create,snapshot mypool/data

# 授予所有权限
zfs allow john all mypool/data
```

#### 12.1.2 查看权限

```bash
# 查看权限
zfs allow mypool/data

# 递归查看
zfs allow -r mypool/data
```

#### 12.1.3 撤销权限

```bash
# 撤销特定权限
zfs unallow john create mypool/data

# 撤销所有权限
zfs unallow john mypool/data

# 递归撤销
zfs unallow -r john mypool/data
```

#### 12.1.4 常用权限

- `create` - 创建数据集
- `destroy` - 销毁数据集
- `mount` - 挂载文件系统
- `snapshot` - 创建快照
- `rollback` - 回滚快照
- `clone` - 创建克隆
- `promote` - 提升克隆
- `send` - 发送数据集
- `receive` - 接收数据集
- `hold` - 保持快照
- `release` - 释放快照
- `compression` - 设置压缩
- `quota` - 设置配额
- `reservation` - 设置预留

### 12.2 ZFS 通道程序

```bash
# 执行 Lua 程序
zfs program mypool script.lua

# 带参数执行
zfs program mypool script.lua arg1 arg2

# 设置内存和指令限制
zfs program -m 10M -t 1000 mypool script.lua
```

示例 Lua 脚本（批量创建快照）：

```lua
-- snapshot-all.lua
args = {...}
snapname = args[1]

for child in zfs.list.children("mypool/data") do
    zfs.sync.snapshot(child .. "@" .. snapname)
end
```

### 12.3 共享功能

#### 12.3.1 NFS 共享

```bash
# 启用 NFS 共享
zfs set sharenfs=on mypool/data

# 带选项共享
zfs set sharenfs="rw,no_root_squash" mypool/data

# 查看共享
zfs get sharenfs mypool/data

# 显示所有共享
zfs share

# 取消共享
zfs set sharenfs=off mypool/data
```

#### 12.3.2 SMB/CIFS 共享

```bash
# 启用 SMB 共享
zfs set sharesmb=on mypool/data

# 带选项共享
zfs set sharesmb="name=myshare" mypool/data

# 取消共享
zfs set sharesmb=off mypool/data
```

### 12.4 块克隆（Block Cloning）

```bash
# 使用 cp --reflink 创建块克隆
cp --reflink=auto source.file dest.file

# 强制使用 reflink
cp --reflink=always source.file dest.file

# 查看块克隆统计
zpool get bcloneused,bclonesaved,bcloneratio mypool
```

### 12.5 去重表管理

```bash
# 查看去重表大小
zpool get dedup_table_size mypool

# 修剪去重表
zpool ddtprune mypool

# 预取去重表到 ARC
zpool prefetch -t ddt mypool
```

### 12.6 等待操作完成

```bash
# 等待删除操作完成
zfs wait -t deleteq mypool/data

# 等待所有活动完成
zpool wait mypool

# 等待特定活动
zpool wait -t scrub mypool
zpool wait -t resilver mypool
zpool wait -t initialize mypool
zpool wait -t trim mypool
```

### 12.7 重写数据

```bash
# 重写数据集以应用新属性
zfs rewrite mypool/data

# 查看重写进度
zpool status -v mypool
```

---

## 13. 最佳实践

### 13.1 池设计建议

#### 13.1.1 VDEV 选择

- **不要混合不同大小的磁盘**在同一 VDEV 中
- **镜像**：适用于需要高性能和冗余的场景
- **RAID-Z2**：最常用的平衡选择（容量、性能、冗余）
- **RAID-Z3**：适用于大磁盘（>4TB）或高可靠性需求
- **dRAID**：适用于大型池（>10 个磁盘）需要快速 resilver

#### 13.1.2 磁盘数量

- **RAID-Z1**：3-5 个磁盘
- **RAID-Z2**：6-10 个磁盘（推荐）
- **RAID-Z3**：8-15 个磁盘
- **镜像**：2 或 3 个磁盘

#### 13.1.3 ashift 设置

```bash
# 对于 4K 扇区磁盘
zpool create -o ashift=12 mypool ...

# 对于 8K 扇区磁盘（某些 SSD）
zpool create -o ashift=13 mypool ...
```

### 13.2 性能最佳实践

#### 13.2.1 通用建议

```bash
# 启用 LZ4 压缩（几乎总是有益）
zfs set compression=lz4 mypool/data

# 禁用 atime
zfs set atime=off mypool/data

# 或使用 relatime
zfs set relatime=on mypool/data

# 启用 autotrim（SSD）
zpool set autotrim=on mypool
```

#### 13.2.2 数据库优化

```bash
# PostgreSQL
zfs create -o recordsize=8K \
    -o primarycache=metadata \
    -o logbias=throughput \
    -o atime=off \
    -o compression=lz4 \
    mypool/postgres

# MySQL/MariaDB
zfs create -o recordsize=16K \
    -o primarycache=metadata \
    -o logbias=throughput \
    -o atime=off \
    -o compression=lz4 \
    mypool/mysql
```

#### 13.2.3 虚拟机优化

```bash
# VM 映像存储
zfs create -V 100G \
    -o volblocksize=16K \
    -o compression=lz4 \
    -o sync=disabled \
    mypool/vm/disk1
```

### 13.3 备份策略

#### 13.3.1 定期快照

```bash
# 使用 cron 或 systemd timer 自动创建快照
# 每小时
0 * * * * zfs snapshot -r mypool/data@hourly-$(date +\%Y\%m\%d-\%H)

# 每天
0 2 * * * zfs snapshot -r mypool/data@daily-$(date +\%Y\%m\%d)

# 每周
0 3 * * 0 zfs snapshot -r mypool/data@weekly-$(date +\%Y\%m\%d)
```

#### 13.3.2 快照保留策略

```bash
# 保留：
# - 24 小时快照
# - 7 天快照
# - 4 周快照
# - 12 月快照

# 自动清理旧快照脚本
#!/bin/bash
# 删除 24 小时前的小时快照
zfs list -H -t snapshot -o name | grep "hourly" | tail -n +25 | xargs -n 1 zfs destroy

# 删除 7 天前的每日快照
zfs list -H -t snapshot -o name | grep "daily" | tail -n +8 | xargs -n 1 zfs destroy
```

#### 13.3.3 远程备份

```bash
#!/bin/bash
# 增量远程备份脚本

SOURCE_POOL="mypool"
SOURCE_DS="data"
REMOTE_HOST="backup.example.com"
REMOTE_POOL="backuppool"
SNAPSHOT_NAME="backup-$(date +%Y%m%d-%H%M%S)"

# 创建快照
zfs snapshot -r ${SOURCE_POOL}/${SOURCE_DS}@${SNAPSHOT_NAME}

# 查找上一个快照
LAST_SNAP=$(zfs list -H -t snapshot -o name -S creation \
    ${SOURCE_POOL}/${SOURCE_DS} | head -2 | tail -1)

# 发送
if [ $(zfs list -H -t snapshot | grep -c ${LAST_SNAP}) -gt 1 ]; then
    # 增量发送
    zfs send -R -i ${LAST_SNAP} ${SOURCE_POOL}/${SOURCE_DS}@${SNAPSHOT_NAME} | \
        pv | \
        ssh ${REMOTE_HOST} "zfs receive -F ${REMOTE_POOL}/${SOURCE_DS}"
else
    # 完整发送
    zfs send -R ${SOURCE_POOL}/${SOURCE_DS}@${SNAPSHOT_NAME} | \
        pv | \
        ssh ${REMOTE_HOST} "zfs receive -F ${REMOTE_POOL}/${SOURCE_DS}"
fi
```

### 13.4 监控建议

#### 13.4.1 关键指标

```bash
# 池健康状态
zpool status -x

# 池容量
zpool list -o name,size,allocated,free,capacity,health

# 碎片率
zpool list -o name,fragmentation

# 压缩比
zfs get compressratio mypool

# 去重比
zfs get dedupratio mypool
```

#### 13.4.2 使用 zed 监控事件

编辑 `/etc/zfs/zed.d/zed.rc`：

```bash
ZED_EMAIL_ADDR="admin@example.com"
ZED_EMAIL_PROG="mail"
ZED_NOTIFY_VERBOSE=1
```

启用 zed：

```bash
systemctl enable zed
systemctl start zed
```

### 13.5 维护建议

#### 13.5.1 定期清洗

```bash
# 每月清洗一次
# 添加到 cron
0 2 1 * * /sbin/zpool scrub mypool
```

#### 13.5.2 检查池健康

```bash
# 每日检查
#!/bin/bash
STATUS=$(zpool status -x)
if [ "$STATUS" != "all pools are healthy" ]; then
    echo "ZFS池有问题！" | mail -s "ZFS 警报" admin@example.com
    echo "$STATUS"
fi
```

#### 13.5.3 监控空间使用

```bash
# 警告阈值：80%
#!/bin/bash
THRESHOLD=80
CAPACITY=$(zpool list -H -o capacity mypool | tr -d '%')

if [ $CAPACITY -gt $THRESHOLD ]; then
    echo "池容量达到 ${CAPACITY}%" | mail -s "ZFS 容量警报" admin@example.com
fi
```

### 13.6 安全建议

#### 13.6.1 使用加密

```bash
# 对敏感数据启用加密
zfs create -o encryption=aes-256-gcm \
    -o keyformat=passphrase \
    mypool/sensitive
```

#### 13.6.2 使用委托管理

```bash
# 不要给普通用户 root 权限
# 使用 zfs allow 授予特定权限
zfs allow developers snapshot,mount,create mypool/dev
```

#### 13.6.3 定期备份

- 使用 `zfs send/receive` 进行远程备份
- 定期测试备份恢复
- 保持异地备份

### 13.7 避免的陷阱

#### 13.7.1 不要

- **不要**在生产环境使用去重（除非有足够内存）
- **不要**使用 `sync=disabled`（除非可以接受数据丢失）
- **不要**在同一 VDEV 混合不同速度的磁盘
- **不要**在没有冗余的情况下使用单盘池
- **不要**使用文件作为生产 VDEV
- **不要**在池接近满的情况下运行（保持 <80%）
- **不要**忘记设置正确的 ashift

#### 13.7.2 应该

- **应该**定期创建快照
- **应该**定期清洗池
- **应该**监控池健康状态
- **应该**使用 ECC 内存（推荐）
- **应该**测试备份恢复
- **应该**记录池配置

### 13.8 容量规划

```bash
# 有效容量计算：

# RAID-Z1: (N-1) * 磁盘大小
# 5 个 2TB 磁盘 = 8TB 有效容量

# RAID-Z2: (N-2) * 磁盘大小
# 6 个 2TB 磁盘 = 8TB 有效容量

# RAID-Z3: (N-3) * 磁盘大小
# 8 个 2TB 磁盘 = 10TB 有效容量

# 镜像: 磁盘大小 / 镜像数
# 2 个 2TB 磁盘镜像 = 2TB 有效容量
```

### 13.9 升级策略

```bash
# 升级前备份池配置
zpool get all mypool > mypool-config-backup.txt

# 导出池配置
zdb -C mypool > mypool-zdb-backup.txt

# 升级前创建快照
zfs snapshot -r mypool@pre-upgrade

# 升级后检查
zpool status
zpool list
zfs mount
```

---

## 附录 A：常用命令速查

### 池管理

```bash
zpool create                 # 创建池
zpool destroy                # 销毁池
zpool import                 # 导入池
zpool export                 # 导出池
zpool list                   # 列出池
zpool status                 # 查看状态
zpool scrub                  # 清洗
zpool add                    # 添加设备
zpool remove                 # 移除设备
zpool replace                # 替换设备
zpool attach                 # 附加设备（镜像）
zpool detach                 # 分离设备
zpool online                 # 上线设备
zpool offline                # 下线设备
zpool clear                  # 清除错误
zpool upgrade                # 升级池
```

### 数据集管理

```bash
zfs create                   # 创建数据集
zfs destroy                  # 销毁数据集
zfs list                     # 列出数据集
zfs rename                   # 重命名
zfs snapshot                 # 创建快照
zfs rollback                 # 回滚快照
zfs clone                    # 克隆快照
zfs promote                  # 提升克隆
zfs send                     # 发送快照
zfs receive                  # 接收快照
zfs mount                    # 挂载
zfs umount                   # 卸载
zfs share                    # 共享
zfs unshare                  # 取消共享
```

### 属性管理

```bash
zfs get                      # 获取属性
zfs set                      # 设置属性
zfs inherit                  # 继承属性
zpool get                    # 获取池属性
zpool set                    # 设置池属性
```

### 其他

```bash
zfs allow                    # 授予权限
zfs unallow                  # 撤销权限
zfs hold                     # 保持快照
zfs release                  # 释放快照
zfs diff                     # 查看差异
zfs bookmark                 # 创建书签
```

---

## 附录 B：重要属性速查

### 池属性

| 属性 | 说明 | 值 |
|------|------|-----|
| autoexpand | 自动扩展 | on/off |
| autoreplace | 自动替换 | on/off |
| autotrim | 自动TRIM | on/off |
| delegation | 启用委托 | on/off |

### 数据集属性

| 属性 | 说明 | 值 |
|------|------|-----|
| compression | 压缩 | on/off/lz4/gzip/zstd |
| checksum | 校验和 | on/off/sha256/sha512 |
| atime | 访问时间 | on/off |
| relatime | 相对atime | on/off |
| quota | 配额 | size/none |
| reservation | 预留 | size/none |
| recordsize | 记录大小 | 512B-16M |
| copies | 副本数 | 1-3 |
| dedup | 去重 | on/off/verify |
| encryption | 加密 | on/off/aes-256-gcm |
| mountpoint | 挂载点 | path/none/legacy |
| readonly | 只读 | on/off |
| sync | 同步写 | standard/always/disabled |

---

## 附录 C：错误代码和消息

### 常见错误

- **EBUSY**: 资源正忙（如快照有克隆）
- **EEXIST**: 对象已存在
- **ENOENT**: 对象不存在
- **ENOSPC**: 空间不足
- **EDQUOT**: 超出配额
- **EROFS**: 只读文件系统

### 池状态消息

- **all pools are healthy**: 所有池正常
- **insufficient replicas**: 副本不足
- **corrupted data**: 数据损坏
- **missing device**: 设备丢失

---

## 附录 D：性能调优参数

### 重要的内核参数

```bash
# ARC 最大值（字节）
zfs_arc_max=8589934592

# ARC 最小值（字节）
zfs_arc_min=1073741824

# 脏数据最大值（字节）
zfs_dirty_data_max=4294967296

# TXG 超时（秒）
zfs_txg_timeout=5

# 预取
zfs_prefetch_disable=0

# 扫描速度限制（字节/秒）
zfs_scan_vdev_limit=4194304
```

---

## 附录 E：故障排查检查清单

### 性能问题

- [ ] 检查池容量（应 <80%）
- [ ] 检查碎片率
- [ ] 检查 ARC 命中率
- [ ] 检查是否启用压缩
- [ ] 检查 recordsize 设置
- [ ] 检查是否有清洗/resilver 运行
- [ ] 检查磁盘 I/O 统计

### 空间问题

- [ ] 检查快照占用空间
- [ ] 检查预留空间
- [ ] 检查隐藏的 .zfs 目录
- [ ] 检查去重表大小
- [ ] 检查 freeing 空间

### 数据完整性

- [ ] 运行 scrub
- [ ] 检查错误计数
- [ ] 检查 SMART 数据
- [ ] 验证备份

---

## 结语

ZFS 是一个功能强大且复杂的文件系统。本手册涵盖了大部分常用操作和最佳实践。建议：

1. **从简单开始**：先熟悉基本操作再使用高级功能
2. **定期备份**：快照不是备份，务必做好异地备份
3. **监控健康状态**：设置自动监控和告警
4. **测试恢复**：定期测试备份恢复流程
5. **阅读文档**：使用 `man zfs` 和 `man zpool` 查看详细文档
6. **谨慎使用高级功能**：如去重、加密等，需充分测试

更多信息：
- 官方文档：https://openzfs.github.io/openzfs-docs/
- 源代码：https://github.com/openzfs/zfs
- 邮件列表：https://openzfs.org/wiki/Mailing_Lists

---

**文档版本**: 1.0
**创建日期**: 2025
**基于源码版本**: OpenZFS (当前代码库)
