# ZFS 存储池安全导出指南

## 目录

1. [概述](#概述)
2. [导出前准备](#导出前准备)
3. [安全导出步骤](#安全导出步骤)
4. [自动化脚本](#自动化脚本)
5. [常见问题解决](#常见问题解决)
6. [重新导入](#重新导入)
7. [特殊场景处理](#特殊场景处理)
8. [最佳实践](#最佳实践)
9. [故障排查检查清单](#故障排查检查清单)

---

## 概述

### 什么是 ZFS 池导出？

导出（export）是将 ZFS 存储池从当前系统中安全移除的过程。导出后：
- 池不再被系统识别和使用
- 所有数据集都会被卸载
- 池可以安全地移动到其他系统
- 池可以重新导入到同一或不同系统

### 为什么需要安全导出？

- **系统维护**：升级硬件、更换服务器
- **数据迁移**：将存储池移动到新系统
- **硬盘维护**：更换或升级磁盘阵列
- **临时移除**：暂时不使用某个池
- **系统关机**：确保数据完整性

### 导出的风险

⚠️ **不正确的导出可能导致**：
- 数据丢失（未同步的数据）
- 文件系统损坏
- 应用程序崩溃（正在使用池的程序）
- 系统不稳定

---

## 导出前准备

### 1. 检查当前状态

#### 查看所有存储池

```bash
# 列出所有池
zpool list

# 示例输出：
# NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH
# mypool    10T   5.2T   4.8T        -         -    15%    52%  1.00x  ONLINE
# backup    5T    2.1T   2.9T        -         -     8%    42%  1.00x  ONLINE
```

#### 查看池的详细状态

```bash
# 查看特定池的状态
zpool status poolname

# 查看所有池的详细状态
zpool status -v

# 示例输出：
#   pool: mypool
#  state: ONLINE
#   scan: scrub repaired 0B in 0h2m with 0 errors on Sun Jan 13 02:00:00 2025
# config:
#
#         NAME        STATE     READ WRITE CKSUM
#         mypool      ONLINE       0     0     0
#           raidz2-0  ONLINE       0     0     0
#             sda     ONLINE       0     0     0
#             sdb     ONLINE       0     0     0
#             sdc     ONLINE       0     0     0
#             sdd     ONLINE       0     0     0
```

#### 查看已挂载的数据集

```bash
# 查看所有已挂载的 ZFS 文件系统
zfs mount

# 查看特定池的挂载点
mount | grep poolname

# 查看数据集列表和挂载点
zfs list -o name,mounted,mountpoint

# 示例输出：
# NAME              MOUNTED  MOUNTPOINT
# mypool            yes      /mypool
# mypool/data       yes      /mypool/data
# mypool/backup     yes      /mypool/backup
```

#### 查看空间使用情况

```bash
# 查看详细的空间使用情况
zfs list -o name,used,available,referenced,mountpoint

# 查看包括快照的使用情况
zfs list -t all -r poolname

# 查看空间分解
zfs list -o space poolname
```

### 2. 检查池健康状态

```bash
# 检查是否有错误
zpool status -x

# 如果所有池正常，会显示：
# all pools are healthy

# 如果有问题，会显示具体的错误信息
```

### 3. 识别依赖关系

#### 检查正在使用池的进程

```bash
# 方法1：使用 lsof
lsof | grep /poolname

# 方法2：使用 fuser
fuser -m /poolname

# 方法3：查看详细信息
fuser -mv /poolname

# 示例输出：
#                      USER        PID ACCESS COMMAND
# /mypool:             root     kernel mount /mypool
#                      postgres   1234 ..c.. postgres
#                      www-data   5678 ..c.. nginx
```

#### 检查挂载的数据集

```bash
# 查看所有 ZFS 挂载
zfs mount | grep poolname

# 查看系统挂载表
df -h | grep poolname
```

#### 检查快照和克隆

```bash
# 列出所有快照
zfs list -t snapshot -r poolname

# 查看克隆依赖
zfs list -o name,origin -t all | grep poolname

# 查看保持的快照
zfs holds -r poolname
```

### 4. 检查共享状态

```bash
# 查看 NFS 共享
zfs get sharenfs poolname
showmount -e localhost

# 查看 SMB 共享
zfs get sharesmb poolname
```

---

## 安全导出步骤

### 步骤 1：创建最后一次快照

这是**最重要**的安全措施，可以在出现问题时恢复数据。

```bash
# 为整个池创建递归快照（包括所有子数据集）
zfs snapshot -r poolname@before-export-$(date +%Y%m%d-%H%M%S)

# 示例：
zfs snapshot -r mypool@before-export-20250113-143000

# 验证快照已创建
zfs list -t snapshot | grep before-export

# 示例输出：
# mypool@before-export-20250113-143000              0B      -    96K  -
# mypool/data@before-export-20250113-143000         0B      -    10G  -
# mypool/backup@before-export-20250113-143000       0B      -     5G  -
```

**为什么要创建快照？**
- 提供数据恢复点
- 记录导出前的确切状态
- 可以用于回滚（如果导出后发现问题）
- 快照创建几乎是瞬时的，不占用额外空间（初始）

### 步骤 2：确保数据完整性

#### 检查池是否有错误

```bash
# 查看错误状态
zpool status -v poolname

# 如果有错误，查看详细信息
zpool status -x

# 如果显示错误，不要继续！先修复
```

#### 可选：运行清洗（Scrub）

```bash
# 开始数据清洗（可选但推荐）
zpool scrub poolname

# 查看清洗进度
watch -n 5 'zpool status poolname'

# 等待清洗完成
# 对于大型池，这可能需要几小时到几天时间

# 如果时间紧迫，可以跳过此步骤
# 但要确保 zpool status 显示 ONLINE 且无错误
```

#### 同步所有数据到磁盘

```bash
# 强制将所有未写入的数据同步到磁盘
zpool sync poolname

# 或同步所有池
zpool sync

# 等待 2-5 秒以确保同步完成
sleep 5

# 这一步非常重要！确保所有缓存数据都写入磁盘
```

### 步骤 3：停止使用池的服务

识别并停止所有使用该池的服务和应用程序。

#### 常见服务示例

```bash
# 数据库服务
sudo systemctl stop postgresql
sudo systemctl stop mysql
sudo systemctl stop mongodb

# Web 服务器
sudo systemctl stop nginx
sudo systemctl stop apache2

# 容器服务
sudo systemctl stop docker
sudo systemctl stop containerd

# 虚拟化服务
sudo systemctl stop libvirtd
sudo systemctl stop qemu-kvm

# 文件共享
sudo systemctl stop smbd
sudo systemctl stop nmbd
sudo systemctl stop nfs-server

# 备份服务
sudo systemctl stop bacula-fd
sudo systemctl stop rsync
```

#### 验证服务已停止

```bash
# 再次检查是否有进程使用池
lsof | grep /poolname
fuser -m /poolname

# 如果还有进程，记录进程信息
ps aux | grep <PID>

# 如果是可以安全终止的进程，手动终止
kill <PID>

# 强制终止（谨慎使用）
kill -9 <PID>
```

### 步骤 4：取消所有共享

```bash
# 取消所有 ZFS 共享（NFS、SMB）
zfs unshare -a

# 或取消特定池的共享
zfs unshare -r poolname

# 验证共享已取消
zfs get sharenfs,sharesmb poolname

# 检查 NFS 导出表
showmount -e localhost

# 检查 Samba 共享
smbclient -L localhost -N
```

### 步骤 5：卸载所有数据集

这是导出前的关键步骤。

#### 方法 1：自动卸载所有（推荐）

```bash
# 卸载所有 ZFS 文件系统
zfs umount -a

# 验证是否已卸载
zfs mount | grep poolname
df -h | grep poolname

# 如果没有输出，说明已成功卸载
```

#### 方法 2：递归卸载特定池

```bash
# 递归卸载特定池的所有数据集
zfs umount -r poolname

# 这会卸载池中的所有文件系统
```

#### 方法 3：逐个卸载

```bash
# 先卸载子数据集，再卸载父数据集
zfs umount poolname/data/documents
zfs umount poolname/data/pictures
zfs umount poolname/data
zfs umount poolname/backup
zfs umount poolname

# 按照从叶子到根的顺序卸载
```

#### 处理卸载失败

```bash
# 如果提示 "device is busy"
# 1. 再次检查占用进程
lsof | grep /poolname
fuser -mv /poolname

# 2. 确认所有服务已停止
systemctl list-units --state=running | grep -i <service>

# 3. 如果确认安全，强制卸载
zfs umount -f poolname/dataset

# 4. 强制卸载所有
zfs umount -f -a

# ⚠️ 强制卸载可能导致数据丢失，仅在确认安全时使用
```

#### 验证卸载完成

```bash
# 检查是否还有挂载的 ZFS 文件系统
zfs mount

# 检查系统挂载表
mount | grep zfs
df -h | grep poolname

# 检查 /proc/mounts
cat /proc/mounts | grep zfs

# 如果这些命令都没有输出，说明卸载成功
```

### 步骤 6：导出存储池

现在可以安全导出池了。

#### 标准导出

```bash
# 导出池
zpool export poolname

# 如果成功，不会有输出
# 验证导出成功
zpool list | grep poolname
# 应该看不到这个池了
```

#### 详细模式导出

```bash
# 使用详细模式查看导出过程
zpool export -v poolname

# 这会显示导出过程中的详细信息
```

#### 强制导出

```bash
# 仅在绝对必要时使用强制导出
zpool export -f poolname

# ⚠️ 警告：强制导出可能导致：
# - 正在写入的数据丢失
# - 文件系统损坏
# - 应用程序崩溃
```

#### 导出失败处理

```bash
# 如果导出失败，查看详细错误
zpool export -v poolname 2>&1 | tee export-error.log

# 常见错误：
# 1. "pool is busy" - 还有文件系统被挂载或进程在使用
# 2. "no such pool" - 池名称错误
# 3. "permission denied" - 需要 root 权限

# 获取详细状态
zpool status -v poolname

# 检查内核日志
dmesg | tail -50
journalctl -xe
```

### 步骤 7：验证导出成功

```bash
# 1. 确认池不在列表中
zpool list
zpool status

# 2. 确认没有挂载点残留
mount | grep poolname
df -h | grep poolname

# 3. 确认没有 /dev 设备
ls -la /dev/zfs
ls -la /dev/zvol/

# 4. 检查系统日志
dmesg | grep zfs | tail -20
journalctl -u zfs-mount -n 50

# 5. 验证可以重新导入（不实际导入）
zpool import
# 应该能看到刚才导出的池
```

---

## 自动化脚本

### 基本导出脚本

```bash
#!/bin/bash
# 文件名: zfs-export.sh
# 用途: 安全导出 ZFS 存储池

set -e  # 遇到错误立即退出

POOL_NAME="$1"

if [ -z "$POOL_NAME" ]; then
    echo "用法: $0 <pool_name>"
    exit 1
fi

echo "开始导出池: $POOL_NAME"

# 1. 检查池是否存在
if ! zpool list "$POOL_NAME" &>/dev/null; then
    echo "错误: 池 '$POOL_NAME' 不存在"
    exit 1
fi

# 2. 同步数据
echo "同步数据到磁盘..."
zpool sync "$POOL_NAME"
sleep 2

# 3. 卸载数据集
echo "卸载所有数据集..."
zfs umount -a

# 4. 导出池
echo "导出池..."
zpool export "$POOL_NAME"

echo "池 '$POOL_NAME' 已成功导出"
```

### 完整的安全导出脚本

```bash
#!/bin/bash
# 文件名: zfs-safe-export.sh
# 用途: 带完整检查和快照的安全导出脚本

set -euo pipefail  # 严格错误处理

# 颜色定义
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 日志函数
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 检查 root 权限
if [ "$EUID" -ne 0 ]; then
    log_error "请使用 root 权限运行此脚本"
    exit 1
fi

# 参数检查
POOL_NAME="${1:-}"
CREATE_SNAPSHOT="${2:-yes}"

if [ -z "$POOL_NAME" ]; then
    echo "用法: $0 <pool_name> [yes|no]"
    echo "  pool_name: 要导出的池名称"
    echo "  yes|no: 是否创建快照（默认: yes）"
    exit 1
fi

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
LOG_FILE="/var/log/zfs-export-${POOL_NAME}-${TIMESTAMP}.log"

# 记录到日志文件
exec > >(tee -a "$LOG_FILE")
exec 2>&1

log_info "=== ZFS 池安全导出流程 ==="
log_info "池名称: $POOL_NAME"
log_info "时间: $TIMESTAMP"
log_info "日志文件: $LOG_FILE"
echo ""

# 步骤 1: 检查池是否存在
log_info "[1/9] 检查池是否存在..."
if ! zpool list "$POOL_NAME" &>/dev/null; then
    log_error "池 '$POOL_NAME' 不存在"
    exit 1
fi
log_info "✓ 池存在"

# 步骤 2: 检查池健康状态
log_info "[2/9] 检查池健康状态..."
HEALTH=$(zpool list -H -o health "$POOL_NAME")
log_info "池健康状态: $HEALTH"

if [ "$HEALTH" != "ONLINE" ]; then
    log_warn "池状态不是 ONLINE"
    zpool status -v "$POOL_NAME"
    read -p "是否继续? (yes/no) " -r
    if [ "$REPLY" != "yes" ]; then
        log_info "用户取消操作"
        exit 1
    fi
fi

# 步骤 3: 创建快照
if [ "$CREATE_SNAPSHOT" = "yes" ]; then
    log_info "[3/9] 创建导出前快照..."
    SNAPSHOT_NAME="${POOL_NAME}@before-export-${TIMESTAMP}"

    if zfs snapshot -r "$SNAPSHOT_NAME"; then
        log_info "✓ 快照创建成功: $SNAPSHOT_NAME"
        echo "$SNAPSHOT_NAME" > "/tmp/zfs-export-snapshot-${POOL_NAME}.txt"
    else
        log_error "快照创建失败"
        read -p "是否继续导出? (yes/no) " -r
        if [ "$REPLY" != "yes" ]; then
            exit 1
        fi
    fi
else
    log_warn "[3/9] 跳过快照创建（用户指定）"
fi

# 步骤 4: 检查使用情况
log_info "[4/9] 检查文件系统使用情况..."
MOUNTPOINT=$(zpool get -H -o value mountpoint "$POOL_NAME" 2>/dev/null || echo "/$POOL_NAME")

if command -v lsof &>/dev/null; then
    USERS=$(lsof 2>/dev/null | grep -c "$MOUNTPOINT" || true)
    if [ "$USERS" -gt 0 ]; then
        log_warn "有 $USERS 个进程正在使用此池"
        log_warn "使用池的进程："
        lsof 2>/dev/null | grep "$MOUNTPOINT" | head -10

        read -p "是否继续? (yes/no) " -r
        if [ "$REPLY" != "yes" ]; then
            log_info "用户取消操作"
            exit 1
        fi
    else
        log_info "✓ 没有进程正在使用此池"
    fi
else
    log_warn "lsof 命令不可用，跳过进程检查"
fi

# 步骤 5: 同步数据
log_info "[5/9] 同步所有数据到磁盘..."
zpool sync "$POOL_NAME"
log_info "等待 5 秒以确保同步完成..."
sleep 5
log_info "✓ 数据同步完成"

# 步骤 6: 取消共享
log_info "[6/9] 取消所有共享..."
if zfs unshare -a 2>/dev/null; then
    log_info "✓ 共享已取消"
else
    log_warn "取消共享时出现警告（可能没有共享）"
fi

# 步骤 7: 检查并记录配置
log_info "[7/9] 保存池配置..."
CONFIG_FILE="/tmp/zfs-export-config-${POOL_NAME}-${TIMESTAMP}.txt"
zpool status -v "$POOL_NAME" > "$CONFIG_FILE"
zfs list -r "$POOL_NAME" >> "$CONFIG_FILE" 2>/dev/null || true
log_info "✓ 配置已保存到: $CONFIG_FILE"

# 步骤 8: 卸载所有数据集
log_info "[8/9] 卸载所有数据集..."

# 先尝试正常卸载
if zfs umount -a 2>/dev/null; then
    log_info "✓ 数据集卸载成功"
else
    log_warn "正常卸载失败，尝试强制卸载..."

    if zfs umount -f -a 2>/dev/null; then
        log_info "✓ 强制卸载成功"
    else
        log_error "卸载失败"
        log_error "请手动检查并停止使用池的进程"
        exit 1
    fi
fi

# 验证卸载
if zfs mount 2>/dev/null | grep -q "$POOL_NAME"; then
    log_error "卸载验证失败，仍有数据集被挂载"
    zfs mount | grep "$POOL_NAME"
    exit 1
fi

# 步骤 9: 导出池
log_info "[9/9] 导出存储池..."

if zpool export "$POOL_NAME"; then
    log_info ""
    log_info "========================================="
    log_info "✓✓✓ 池 '$POOL_NAME' 已成功导出 ✓✓✓"
    log_info "========================================="
    log_info "导出时间: $TIMESTAMP"
    [ "$CREATE_SNAPSHOT" = "yes" ] && log_info "最后快照: $SNAPSHOT_NAME"
    log_info "配置备份: $CONFIG_FILE"
    log_info "日志文件: $LOG_FILE"
    log_info ""
    log_info "重新导入命令:"
    log_info "  zpool import $POOL_NAME"
    log_info ""
else
    log_error ""
    log_error "========================================="
    log_error "✗✗✗ 导出失败 ✗✗✗"
    log_error "========================================="
    log_error "请查看上面的错误信息"
    log_error "日志文件: $LOG_FILE"

    # 尝试获取详细错误
    log_info ""
    log_info "尝试获取详细错误信息..."
    zpool export -v "$POOL_NAME" 2>&1 || true

    exit 1
fi
```

### 使用脚本

```bash
# 1. 保存脚本
sudo nano /usr/local/bin/zfs-safe-export.sh

# 2. 添加执行权限
sudo chmod +x /usr/local/bin/zfs-safe-export.sh

# 3. 运行脚本
sudo /usr/local/bin/zfs-safe-export.sh mypool

# 4. 不创建快照的方式运行
sudo /usr/local/bin/zfs-safe-export.sh mypool no

# 5. 查看日志
tail -f /var/log/zfs-export-mypool-*.log
```

### 批量导出脚本

```bash
#!/bin/bash
# 文件名: zfs-export-all.sh
# 用途: 导出所有非根池

set -euo pipefail

log_info() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [INFO] $1"
}

log_error() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [ERROR] $1" >&2
}

# 获取所有池（排除根池）
POOLS=$(zpool list -H -o name | grep -v "^rpool$" | grep -v "^bpool$")

if [ -z "$POOLS" ]; then
    log_info "没有需要导出的池"
    exit 0
fi

log_info "找到以下池："
echo "$POOLS"
echo ""

read -p "确定要导出这些池吗? (yes/no) " -r
if [ "$REPLY" != "yes" ]; then
    log_info "用户取消操作"
    exit 0
fi

# 逐个导出
for POOL in $POOLS; do
    log_info "=========================================="
    log_info "开始导出池: $POOL"
    log_info "=========================================="

    if /usr/local/bin/zfs-safe-export.sh "$POOL"; then
        log_info "✓ 池 '$POOL' 导出成功"
    else
        log_error "✗ 池 '$POOL' 导出失败"
        read -p "是否继续导出其他池? (yes/no) " -r
        if [ "$REPLY" != "yes" ]; then
            exit 1
        fi
    fi

    echo ""
done

log_info "所有池导出完成"
```

---

## 常见问题解决

### 问题 1：设备忙（Device is busy）

**症状**：
```
cannot unmount '/poolname/data': Device busy
```

**原因**：
- 有进程正在使用文件系统
- 有打开的文件
- 当前工作目录在该文件系统中

**解决方案**：

#### 步骤 1：找出占用进程

```bash
# 方法 1: 使用 lsof（最详细）
lsof | grep /poolname

# 示例输出：
# nginx     1234  www-data    5r   REG  0,123    12345  /poolname/data/file.txt
# postgres  5678  postgres   10u   REG  0,123    67890  /poolname/data/db/data.db

# 方法 2: 使用 fuser
fuser -m /poolname

# 方法 3: 使用 fuser 显示详细信息
fuser -mv /poolname

# 示例输出：
#                      USER        PID ACCESS COMMAND
# /poolname:           root     kernel mount /poolname
#                      postgres   1234 ..c.. postgres
#                      www-data   5678 F.... nginx
```

#### 步骤 2：识别进程类型

**ACCESS 列含义**：
- `c` : 当前目录
- `e` : 可执行文件
- `f` : 打开的文件
- `F` : 打开的文件（写入模式）
- `r` : 根目录
- `m` : mmap 文件

#### 步骤 3：停止占用进程

```bash
# 如果是服务
sudo systemctl stop <service-name>

# 如果是普通进程
sudo kill <PID>

# 如果进程不响应
sudo kill -9 <PID>

# 批量终止所有占用进程（危险！）
sudo fuser -km /poolname

# ⚠️ 这会强制终止所有使用该文件系统的进程
```

#### 步骤 4：检查当前目录

```bash
# 如果你的 shell 在该目录中
pwd | grep /poolname

# 切换到其他目录
cd /

# 检查所有用户的当前目录
lsof | grep cwd | grep /poolname
```

#### 步骤 5：检查 NFS 导出

```bash
# 查看 NFS 导出
exportfs -v

# 取消 NFS 导出
sudo exportfs -u :/poolname

# 或重启 NFS 服务
sudo systemctl restart nfs-server
```

#### 步骤 6：强制卸载

```bash
# 仅在确认安全后使用
zfs umount -f /poolname/dataset

# ⚠️ 警告：可能导致数据丢失
```

### 问题 2：数据集无法卸载

**症状**：
```
cannot unmount '/poolname/data': umount failed
```

**解决方案**：

```bash
# 1. 检查挂载状态
mount | grep /poolname
zfs mount | grep /poolname

# 2. 检查是否有嵌套挂载
mount | grep /poolname | sort

# 3. 从最深层开始卸载
# 先卸载: /poolname/data/subdir
# 再卸载: /poolname/data
# 最后卸载: /poolname

# 4. 检查是否有快照挂载
ls -la /poolname/.zfs/snapshot/

# 5. 如果有克隆，先处理克隆
zfs list -o name,origin | grep /poolname

# 6. 使用 umount 命令
sudo umount /poolname/data
```

### 问题 3：池显示为 UNAVAIL 或 FAULTED

**症状**：
```
  pool: mypool
 state: FAULTED
```

**解决方案**：

```bash
# 1. 查看详细状态
zpool status -v poolname

# 2. 不要强制导出！先尝试修复

# 3. 清除错误
zpool clear poolname

# 4. 如果是磁盘问题，替换磁盘
zpool replace poolname /dev/old-disk /dev/new-disk

# 5. 等待 resilver 完成
watch -n 5 'zpool status poolname'

# 6. 如果有检查点，可以回滚
zpool import --rewind-to-checkpoint poolname

# 7. 只读导入（最后手段）
zpool import -o readonly=on poolname
# 备份数据后再处理
```

### 问题 4：导出后无法重新导入

**症状**：
```
cannot import 'poolname': no such pool available
```

**解决方案**：

```bash
# 1. 搜索可导入的池
zpool import

# 2. 指定搜索路径
zpool import -d /dev/disk/by-id

# 3. 使用池 ID 导入
zpool import <pool-id>

# 4. 强制导入
zpool import -f poolname

# 5. 扫描特定目录
zpool import -d /dev

# 6. 显示详细信息
zpool import -v

# 7. 恢复已销毁的池
zpool import -D
```

### 问题 5：权限被拒绝

**症状**：
```
cannot export 'poolname': permission denied
```

**解决方案**：

```bash
# 1. 使用 root 权限
sudo zpool export poolname

# 2. 检查当前用户
whoami
id

# 3. 检查 ZFS 权限
zfs allow poolname

# 4. 检查 SELinux（RHEL/CentOS）
getenforce
sudo setenforce 0  # 临时禁用

# 5. 检查 AppArmor（Ubuntu）
sudo aa-status
```

### 问题 6：快照有克隆依赖

**症状**：
```
cannot destroy 'poolname@snap': snapshot has dependent clones
```

**解决方案**：

```bash
# 1. 查看克隆关系
zfs list -o name,origin | grep poolname@snap

# 2. 提升克隆（使其独立）
zfs promote poolname/clone

# 3. 或先销毁克隆
zfs destroy poolname/clone

# 4. 查看所有依赖
zfs get clones poolname@snap

# 5. 递归销毁（危险）
zfs destroy -r poolname@snap
```

### 问题 7：zvol 正在使用

**症状**：
```
cannot export 'poolname': pool is busy
```

**解决方案**：

```bash
# 1. 检查 zvol 使用情况
ls -la /dev/zvol/poolname/

# 2. 检查是否被虚拟机使用
virsh list --all
virsh domblklist <vm-name>

# 3. 停止虚拟机
virsh shutdown <vm-name>
virsh destroy <vm-name>  # 强制停止

# 4. 检查 iSCSI target
tgtadm --mode target --op show

# 5. 停止 iSCSI target
sudo systemctl stop tgt

# 6. 检查设备映射
dmsetup ls
dmsetup remove <device>
```

---

## 重新导入

导出的池可以重新导入到同一系统或不同系统。

### 基本导入

```bash
# 1. 列出可导入的池
zpool import

# 示例输出：
#    pool: mypool
#      id: 1234567890
#   state: ONLINE
#  action: The pool can be imported using its name or numeric identifier.
#  config:
#         mypool      ONLINE
#           raidz2-0  ONLINE
#             sda     ONLINE
#             sdb     ONLINE

# 2. 使用池名导入
zpool import mypool

# 3. 使用数字 ID 导入
zpool import 1234567890

# 4. 验证导入
zpool status mypool
zfs list -r mypool
```

### 高级导入选项

#### 使用不同名称导入

```bash
# 导入时重命名
zpool import oldname newname

# 适用场景：
# - 避免名称冲突
# - 测试环境使用不同名称
```

#### 使用备用根目录导入

```bash
# 导入到备用根目录
zpool import -R /mnt mypool

# 所有挂载点都会以 /mnt 为前缀
# 例如: /mypool/data -> /mnt/mypool/data

# 适用场景：
# - 数据恢复
# - 检查池内容而不影响原系统
# - 临时访问
```

#### 只读导入

```bash
# 只读模式导入
zpool import -o readonly=on mypool

# 适用场景：
# - 数据恢复
# - 检查数据而不修改
# - 防止意外修改
```

#### 强制导入

```bash
# 强制导入（之前在其他系统使用）
zpool import -f mypool

# 适用场景：
# - 从另一系统移动池
# - 系统崩溃后恢复
# - 池显示为 "in use"

# ⚠️ 警告：
# - 确保池未在其他系统使用
# - 可能导致数据损坏如果池仍在其他系统使用
```

#### 指定搜索目录

```bash
# 在特定目录搜索设备
zpool import -d /dev/disk/by-id

# 搜索多个目录
zpool import -d /dev/disk/by-id -d /dev/disk/by-uuid

# 适用场景：
# - 设备路径变化
# - 使用特定设备标识
```

#### 恢复已销毁的池

```bash
# 列出可恢复的已销毁池
zpool import -D

# 导入已销毁的池
zpool import -D poolname

# ⚠️ 限制：
# - 仅在导出后立即有效
# - 不保证数据完整性
# - 应尽快备份数据
```

#### 回滚到检查点

```bash
# 导入并回滚到检查点
zpool import --rewind-to-checkpoint mypool

# 适用场景：
# - 撤销最近的更改
# - 恢复到已知良好状态
```

#### 回滚到特定事务组

```bash
# 查看可回滚的事务组
zpool import -F mypool

# 回滚到特定 TXG
zpool import -F -T <txg> mypool

# 适用场景：
# - 高级数据恢复
# - 池损坏恢复
```

### 导入验证

```bash
# 1. 检查池状态
zpool status mypool

# 2. 验证数据完整性
zpool scrub mypool

# 3. 检查挂载
zfs mount
df -h | grep mypool

# 4. 测试访问
ls -la /mypool
cd /mypool && ls

# 5. 检查快照是否存在
zfs list -t snapshot -r mypool

# 6. 验证之前的快照
zfs list -t snapshot | grep before-export
```

### 导入后恢复服务

```bash
# 1. 启动数据库服务
sudo systemctl start postgresql
sudo systemctl start mysql

# 2. 启动 Web 服务
sudo systemctl start nginx

# 3. 重新启用共享
zfs set sharenfs=on mypool/shared
zfs share -a

# 4. 验证应用程序
# 检查数据库连接
# 检查 Web 应用
# 验证备份任务
```

---

## 特殊场景处理

### 场景 1：系统根目录在 ZFS 上

⚠️ **绝对不要导出根池！**

**识别根池**：

```bash
# 检查根文件系统
df -h /

# 检查 ZFS 挂载
zfs list | grep " /$"

# 常见根池名称: rpool, bpool
```

**如果需要迁移根池**：

1. 使用 Live CD/USB 启动系统
2. 导入池到临时位置
3. 备份数据
4. 迁移到新系统

```bash
# 从 Live CD 操作
zpool import -R /mnt rpool
rsync -avP /mnt/ /new-root/
```

### 场景 2：加密的池

导出加密池与普通池类似，但重新导入时需要额外步骤。

#### 导出加密池

```bash
# 1. 创建快照（包含加密状态）
zfs snapshot -r poolname@before-export

# 2. 正常导出
zpool export poolname

# 导出时密钥会自动卸载
```

#### 导入加密池

```bash
# 1. 导入池
zpool import poolname

# 2. 加载密钥
zfs load-key poolname/encrypted-dataset

# 或批量加载
zfs load-key -r poolname

# 3. 挂载数据集
zfs mount poolname/encrypted-dataset

# 或挂载所有
zfs mount -a
```

#### 加密池完整示例

```bash
# 导入并自动挂载
zpool import poolname && \
zfs load-key -a && \
zfs mount -a

# 使用密钥文件
zfs load-key -L file:///root/zfs-key poolname/encrypted

# 验证密钥状态
zfs get keystatus -r poolname
```

### 场景 3：有活动的 zvol（块设备）

zvol 常用于虚拟机磁盘、iSCSI target 等。

#### 识别 zvol

```bash
# 列出所有 zvol
zfs list -t volume

# 检查 zvol 设备
ls -la /dev/zvol/poolname/

# 检查 zvol 使用情况
lsblk | grep zvol
```

#### 停止 zvol 使用

```bash
# 1. 停止虚拟机
virsh list --all
virsh shutdown vm-name
virsh destroy vm-name  # 强制停止

# 2. 停止 iSCSI target
sudo systemctl stop tgt
# 或
sudo systemctl stop iscsi

# 3. 卸载文件系统（如果 zvol 作为块设备挂载）
umount /dev/zvol/poolname/volume

# 4. 停止 LVM（如果在 zvol 上）
vgchange -an vg-name

# 5. 关闭加密设备（如果使用 LUKS）
cryptsetup close encrypted-zvol
```

#### 导出带 zvol 的池

```bash
# 1. 确认没有活动使用
lsof | grep zvol

# 2. 正常导出
zpool export poolname

# zvol 设备会自动消失
ls /dev/zvol/  # 应该看不到了
```

### 场景 4：有 NFS/SMB 共享的池

#### 导出前处理

```bash
# 1. 列出所有 NFS 共享
showmount -e localhost
exportfs -v

# 2. 列出所有 SMB 共享
smbclient -L localhost -N

# 3. 检查 ZFS 共享配置
zfs get sharenfs,sharesmb -r poolname

# 4. 取消所有共享
zfs unshare -a

# 5. 停止共享服务（可选）
sudo systemctl stop nfs-server
sudo systemctl stop smbd nmbd

# 6. 验证共享已取消
showmount -e localhost
# 应该看不到 poolname 的导出
```

#### 导入后恢复共享

```bash
# 1. 导入池
zpool import poolname

# 2. 检查共享配置是否保留
zfs get sharenfs,sharesmb -r poolname

# 3. 启动共享服务
sudo systemctl start nfs-server
sudo systemctl start smbd nmbd

# 4. 手动共享（如果需要）
zfs set sharenfs=on poolname/shared
zfs set sharesmb=on poolname/smb-share

# 5. 或批量共享
zfs share -a

# 6. 验证
showmount -e localhost
smbclient -L localhost -N
```

### 场景 5：在容器中使用的池

#### Docker 容器

```bash
# 1. 列出使用 ZFS 存储的容器
docker ps -a --filter volume=/poolname

# 2. 停止容器
docker stop container-name

# 3. 停止 Docker 服务
sudo systemctl stop docker

# 4. 等待所有容器完全停止
sleep 10

# 5. 验证没有 Docker 进程
ps aux | grep docker

# 6. 导出池
zpool export poolname
```

#### LXC/LXD 容器

```bash
# 1. 列出容器
lxc list

# 2. 停止容器
lxc stop container-name

# 3. 或停止所有容器
lxc stop --all

# 4. 停止 LXD 服务
sudo systemctl stop lxd

# 5. 导出池
zpool export poolname
```

### 场景 6：有正在运行的清洗或 resilver

```bash
# 1. 检查清洗/resilver 状态
zpool status poolname

# 示例输出：
#   scan: scrub in progress since Sun Jan 13 14:00:00 2025
#         5.2T scanned at 1.2G/s, 2.1T issued at 500M/s
#         0B repaired, 40.38% done, 01:23:45 to go

# 2. 选项 A: 等待完成（推荐）
watch -n 5 'zpool status poolname'

# 3. 选项 B: 停止清洗
zpool scrub -s poolname

# 4. 等待操作停止
sleep 5

# 5. 验证已停止
zpool status poolname

# 6. 导出池
zpool export poolname
```

### 场景 7：使用 ZFS 委托权限

```bash
# 1. 检查委托的权限
zfs allow poolname

# 2. 记录权限配置
zfs allow poolname > /tmp/zfs-permissions-backup.txt

# 3. 导出池（权限会保留）
zpool export poolname

# 4. 导入池（权限自动恢复）
zpool import poolname

# 5. 验证权限
zfs allow poolname
```

---

## 最佳实践

### 导出前

✅ **始终要做的**：

1. **创建快照**
   ```bash
   zfs snapshot -r poolname@before-export-$(date +%Y%m%d-%H%M%S)
   ```

2. **备份池配置**
   ```bash
   zpool status -v poolname > /backup/pool-config-$(date +%Y%m%d).txt
   zfs list -r poolname >> /backup/pool-config-$(date +%Y%m%d).txt
   ```

3. **同步数据**
   ```bash
   zpool sync poolname
   sleep 5
   ```

4. **检查健康状态**
   ```bash
   zpool status -x
   ```

5. **记录重要信息**
   ```bash
   # 池 GUID
   zpool get guid poolname

   # 数据集列表
   zfs list -r poolname

   # 快照列表
   zfs list -t snapshot -r poolname
   ```

### 导出过程中

✅ **推荐做法**：

1. **使用脚本**：自动化减少错误
2. **记录日志**：便于问题排查
3. **逐步验证**：每步都验证成功
4. **避免强制**：不要轻易使用 `-f` 选项

⚠️ **避免的错误**：

1. ❌ 不检查就强制导出
2. ❌ 跳过数据同步
3. ❌ 不创建快照
4. ❌ 在有错误时导出
5. ❌ 不记录配置

### 导出后

✅ **验证清单**：

- [ ] 池不在 `zpool list` 中
- [ ] 没有残留挂载点
- [ ] `/dev/zvol/` 中没有设备
- [ ] 可以看到池在 `zpool import` 中
- [ ] 日志没有错误信息
- [ ] 配置已备份
- [ ] 快照已创建

### 文档记录

**每次导出都应记录**：

```bash
# 创建导出报告
cat > /var/log/zfs-export-report-$(date +%Y%m%d).txt <<EOF
ZFS 池导出报告
===============
日期: $(date)
操作员: $(whoami)
主机: $(hostname)

池信息:
$(zpool list poolname)

池配置:
$(zpool status -v poolname)

数据集:
$(zfs list -r poolname)

快照:
$(zfs list -t snapshot -r poolname | tail -5)

导出原因: [填写原因]

验证结果: [成功/失败]

备注: [其他信息]
EOF
```

### 定期维护

**在导出前进行的定期维护**：

```bash
# 1. 每月一次清洗
zpool scrub poolname

# 2. 检查池健康
zpool status -v poolname

# 3. 清理旧快照
# （根据保留策略）

# 4. 检查碎片率
zpool list -o name,fragmentation poolname

# 5. 验证备份
# 确保有异地备份
```

---

## 故障排查检查清单

### 导出失败排查

当导出失败时，按以下顺序检查：

#### ☑️ 第 1 步：检查池状态

```bash
# 池是否存在？
zpool list | grep poolname

# 池是否健康？
zpool status poolname

# 是否有错误？
zpool status -x
```

**预期结果**：
- 池存在
- 状态为 ONLINE
- 没有错误

#### ☑️ 第 2 步：检查挂载状态

```bash
# 是否有挂载的数据集？
zfs mount | grep poolname
mount | grep poolname
df -h | grep poolname

# 尝试卸载
zfs umount -a
```

**预期结果**：
- 所有数据集都已卸载
- `zfs mount` 中看不到池

#### ☑️ 第 3 步：检查进程占用

```bash
# 是否有进程在使用？
lsof | grep /poolname
fuser -mv /poolname

# 检查当前目录
pwd | grep /poolname
```

**预期结果**：
- 没有进程使用
- 不在池的目录中

#### ☑️ 第 4 步：检查服务

```bash
# 检查常见服务
systemctl status postgresql | grep active
systemctl status mysql | grep active
systemctl status docker | grep active
systemctl status nfs-server | grep active
```

**预期结果**：
- 使用池的服务都已停止

#### ☑️ 第 5 步：检查共享

```bash
# NFS 共享
showmount -e localhost | grep poolname
exportfs -v | grep poolname

# SMB 共享
smbstatus | grep poolname
```

**预期结果**：
- 没有活动共享

#### ☑️ 第 6 步：检查 zvol

```bash
# zvol 设备
ls /dev/zvol/poolname/

# zvol 使用
lsblk | grep zvol
dmsetup ls | grep zvol
```

**预期结果**：
- zvol 没有被使用

#### ☑️ 第 7 步：检查系统日志

```bash
# 查看最近的错误
dmesg | grep -i zfs | tail -50
journalctl -xe | grep -i zfs
cat /var/log/syslog | grep -i zfs | tail -50
```

**预期结果**：
- 没有致命错误

#### ☑️ 第 8 步：尝试详细导出

```bash
# 使用详细模式
zpool export -v poolname 2>&1 | tee export-error.log

# 查看具体错误
cat export-error.log
```

### 快速诊断脚本

```bash
#!/bin/bash
# 文件名: zfs-export-diagnose.sh
# 用途: 诊断为什么池无法导出

POOL_NAME="$1"

if [ -z "$POOL_NAME" ]; then
    echo "用法: $0 <pool_name>"
    exit 1
fi

echo "=== ZFS 池导出诊断 ==="
echo "池名称: $POOL_NAME"
echo "时间: $(date)"
echo ""

# 1. 检查池是否存在
echo "[检查 1] 池是否存在..."
if zpool list "$POOL_NAME" &>/dev/null; then
    echo "✓ 池存在"
    HEALTH=$(zpool list -H -o health "$POOL_NAME")
    echo "  状态: $HEALTH"
else
    echo "✗ 池不存在"
    exit 1
fi
echo ""

# 2. 检查挂载
echo "[检查 2] 检查挂载状态..."
MOUNTED=$(zfs mount 2>/dev/null | grep "$POOL_NAME" | wc -l)
if [ "$MOUNTED" -eq 0 ]; then
    echo "✓ 没有挂载的数据集"
else
    echo "✗ 有 $MOUNTED 个数据集被挂载:"
    zfs mount | grep "$POOL_NAME"
fi
echo ""

# 3. 检查进程占用
echo "[检查 3] 检查进程占用..."
if command -v lsof &>/dev/null; then
    PROCS=$(lsof 2>/dev/null | grep -c "/$POOL_NAME" || echo 0)
    if [ "$PROCS" -eq 0 ]; then
        echo "✓ 没有进程占用"
    else
        echo "✗ 有 $PROCS 个进程正在使用:"
        lsof 2>/dev/null | grep "/$POOL_NAME" | head -5
    fi
else
    echo "⚠ lsof 不可用，跳过检查"
fi
echo ""

# 4. 检查 zvol
echo "[检查 4] 检查 zvol..."
if [ -d "/dev/zvol/$POOL_NAME" ]; then
    ZVOLS=$(ls "/dev/zvol/$POOL_NAME" 2>/dev/null | wc -l)
    if [ "$ZVOLS" -eq 0 ]; then
        echo "✓ 没有活动的 zvol"
    else
        echo "⚠ 有 $ZVOLS 个 zvol 设备"
        ls -la "/dev/zvol/$POOL_NAME"
    fi
else
    echo "✓ 没有 zvol"
fi
echo ""

# 5. 检查共享
echo "[检查 5] 检查共享状态..."
SHARES=$(zfs get -H sharenfs,sharesmb -r "$POOL_NAME" 2>/dev/null | grep -v "off" | grep -v "sharenfs.*-" | wc -l)
if [ "$SHARES" -eq 0 ]; then
    echo "✓ 没有活动共享"
else
    echo "⚠ 有共享配置:"
    zfs get sharenfs,sharesmb -r "$POOL_NAME" | grep -v "off"
fi
echo ""

# 6. 检查错误
echo "[检查 6] 检查池错误..."
ERRORS=$(zpool status "$POOL_NAME" | grep -i "errors:" | awk '{print $2}')
if [ "$ERRORS" = "No" ] || [ "$ERRORS" = "0" ]; then
    echo "✓ 没有错误"
else
    echo "✗ 发现错误:"
    zpool status -v "$POOL_NAME"
fi
echo ""

# 7. 尝试诊断导出
echo "[检查 7] 尝试导出..."
if zpool export "$POOL_NAME" 2>/dev/null; then
    echo "✓✓✓ 导出成功！"
    echo ""
    echo "池已被导出。使用以下命令重新导入:"
    echo "  zpool import $POOL_NAME"
else
    echo "✗ 导出失败"
    echo ""
    echo "详细错误:"
    zpool export -v "$POOL_NAME" 2>&1
fi
echo ""

# 总结
echo "=== 诊断完成 ==="
echo ""
echo "如果导出失败，请按以下步骤操作:"
echo "1. 卸载所有数据集: zfs umount -a"
echo "2. 停止占用进程"
echo "3. 取消共享: zfs unshare -a"
echo "4. 同步数据: zpool sync $POOL_NAME"
echo "5. 再次尝试导出: zpool export $POOL_NAME"
```

### 使用诊断脚本

```bash
# 保存脚本
sudo nano /usr/local/bin/zfs-export-diagnose.sh

# 添加执行权限
sudo chmod +x /usr/local/bin/zfs-export-diagnose.sh

# 运行诊断
sudo /usr/local/bin/zfs-export-diagnose.sh mypool
```

---

## 附录 A：命令速查表

### 导出前检查

| 命令 | 说明 |
|------|------|
| `zpool list` | 列出所有池 |
| `zpool status poolname` | 查看池状态 |
| `zfs mount` | 查看已挂载数据集 |
| `lsof \| grep /poolname` | 查看使用池的进程 |
| `fuser -mv /poolname` | 详细查看占用情况 |
| `zfs get sharenfs,sharesmb -r poolname` | 查看共享配置 |

### 准备导出

| 命令 | 说明 |
|------|------|
| `zfs snapshot -r poolname@name` | 创建递归快照 |
| `zpool sync poolname` | 同步数据到磁盘 |
| `zfs unshare -a` | 取消所有共享 |
| `systemctl stop <service>` | 停止服务 |

### 导出操作

| 命令 | 说明 |
|------|------|
| `zfs umount -a` | 卸载所有数据集 |
| `zfs umount -r poolname` | 递归卸载池 |
| `zpool export poolname` | 导出池 |
| `zpool export -v poolname` | 详细导出 |
| `zpool export -f poolname` | 强制导出（危险） |

### 导入操作

| 命令 | 说明 |
|------|------|
| `zpool import` | 列出可导入池 |
| `zpool import poolname` | 导入池 |
| `zpool import -f poolname` | 强制导入 |
| `zpool import -R /mnt poolname` | 使用备用根目录 |
| `zpool import -o readonly=on poolname` | 只读导入 |
| `zpool import -D` | 列出可恢复的已销毁池 |

### 验证

| 命令 | 说明 |
|------|------|
| `zpool list \| grep poolname` | 验证池已导出 |
| `mount \| grep poolname` | 检查残留挂载 |
| `zpool status poolname` | 检查池状态（导入后） |

---

## 附录 B：错误代码和消息

### 常见错误消息

| 错误消息 | 原因 | 解决方案 |
|---------|------|---------|
| `pool is busy` | 有资源正在使用 | 检查并停止占用进程 |
| `device is busy` | 文件系统被占用 | 卸载数据集或停止进程 |
| `umount failed` | 卸载失败 | 检查占用，使用 `-f` 强制 |
| `permission denied` | 权限不足 | 使用 sudo |
| `no such pool` | 池不存在或已导出 | 检查池名称 |
| `cannot import` | 导入失败 | 使用 `-f` 或检查设备 |
| `pool was destroyed` | 池已被销毁 | 使用 `-D` 恢复 |
| `pool is FAULTED` | 池故障 | 先修复再导出 |

---

## 附录 C：安全检查清单

### 导出前检查清单

在导出池之前，确保完成以下检查：

- [ ] 已创建最后一次快照
- [ ] 已备份重要数据
- [ ] 已记录池配置
- [ ] 池健康状态为 ONLINE
- [ ] 没有正在运行的清洗或 resilver
- [ ] 已停止所有使用池的服务
- [ ] 所有数据已同步到磁盘
- [ ] 已取消所有 NFS/SMB 共享
- [ ] 没有进程占用池
- [ ] 不是系统根池
- [ ] 已卸载所有数据集
- [ ] 已记录导出原因和时间

### 导出后验证清单

导出后，验证以下项目：

- [ ] 池不在 `zpool list` 输出中
- [ ] 没有残留挂载点
- [ ] `/dev/zvol/` 中没有池的设备
- [ ] 池出现在 `zpool import` 列表中
- [ ] 系统日志没有错误
- [ ] 配置文件已保存
- [ ] 快照信息已记录
- [ ] 可以成功重新导入（如果需要）

---

## 结语

安全导出 ZFS 存储池是一个需要谨慎操作的过程。遵循本指南的步骤可以最大程度地降低数据丢失风险。

### 关键要点

1. **永远创建快照** - 这是你的安全网
2. **检查再检查** - 导出前确认所有依赖都已处理
3. **同步数据** - 确保所有写入完成
4. **记录一切** - 便于问题排查和审计
5. **测试导入** - 验证池可以重新导入

### 寻求帮助

如果遇到问题：

1. 查看系统日志：`dmesg`, `journalctl`
2. 运行诊断脚本
3. 参考 OpenZFS 文档
4. 咨询 ZFS 社区：
   - 邮件列表：https://openzfs.org/wiki/Mailing_Lists
   - IRC: #openzfs on Libera.Chat
   - Reddit: r/zfs

### 相关文档

- [ZFS 操作手册](./ZFS操作手册.md)
- [OpenZFS 官方文档](https://openzfs.github.io/openzfs-docs/)
- `man zpool`
- `man zfs`

---

**文档版本**: 1.0
**最后更新**: 2025-01-13
**作者**: ZFS 管理员指南项目
**许可**: CDDL-1.0
