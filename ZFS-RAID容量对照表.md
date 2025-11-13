# ZFS RAID 配置容量对照表

## 目录

1. [容量计算公式](#容量计算公式)
2. [镜像配置容量表](#镜像配置容量表)
3. [RAID-Z1 容量表](#raid-z1-容量表)
4. [RAID-Z2 容量表](#raid-z2-容量表)
5. [RAID-Z3 容量表](#raid-z3-容量表)
6. [dRAID 容量表](#draid-容量表)
7. [常见磁盘容量对照](#常见磁盘容量对照)
8. [容量计算器](#容量计算器)
9. [配置推荐](#配置推荐)

---

## 容量计算公式

### 基本公式

#### 镜像（Mirror）

```
可用容量 = 磁盘大小 / 镜像数
冗余盘数 = 镜像数 - 1
```

**示例**：
- 2 盘镜像：可用 50%（1/2）
- 3 盘镜像：可用 33%（1/3）

#### RAID-Z1（单奇偶校验）

```
可用容量 = (磁盘数 - 1) × 磁盘大小
可用比例 = (N - 1) / N
冗余盘数 = 1
```

**示例**：
- 3 盘 RAID-Z1：可用 67%（2/3）
- 5 盘 RAID-Z1：可用 80%（4/5）

#### RAID-Z2（双奇偶校验）

```
可用容量 = (磁盘数 - 2) × 磁盘大小
可用比例 = (N - 2) / N
冗余盘数 = 2
```

**示例**：
- 4 盘 RAID-Z2：可用 50%（2/4）
- 6 盘 RAID-Z2：可用 67%（4/6）

#### RAID-Z3（三重奇偶校验）

```
可用容量 = (磁盘数 - 3) × 磁盘大小
可用比例 = (N - 3) / N
冗余盘数 = 3
```

**示例**：
- 5 盘 RAID-Z3：可用 40%（2/5）
- 8 盘 RAID-Z3：可用 63%（5/8）

#### dRAID（分布式 RAID）

```
可用容量 ≈ (磁盘数 - 热备数) × (D / (D + P)) × 磁盘大小
可用比例 ≈ ((N - S) / N) × (D / (D + P))

其中：
N = 总磁盘数
S = 热备盘数
D = 每组数据盘数（默认 8）
P = 奇偶校验盘数（1/2/3 对应 draid1/2/3）
```

### 实际可用空间说明

⚠️ **注意**：实际可用空间会略小于理论值，原因：
- ZFS 元数据开销（约 1-2%）
- 磁盘容量标注差异（1000 vs 1024 进制）
- 对齐和填充开销
- 实际通常为理论值的 95-98%

---

## 镜像配置容量表

### 双盘镜像（2-Way Mirror）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 |
|--------|----------|-----------|----------|--------|-----------|
| 2      | 1TB      | 2TB       | 1TB      | 50%    | 1         |
| 2      | 2TB      | 4TB       | 2TB      | 50%    | 1         |
| 2      | 4TB      | 8TB       | 4TB      | 50%    | 1         |
| 2      | 6TB      | 12TB      | 6TB      | 50%    | 1         |
| 2      | 8TB      | 16TB      | 8TB      | 50%    | 1         |
| 2      | 10TB     | 20TB      | 10TB     | 50%    | 1         |
| 2      | 12TB     | 24TB      | 12TB     | 50%    | 1         |
| 2      | 16TB     | 32TB      | 16TB     | 50%    | 1         |
| 2      | 20TB     | 40TB      | 20TB     | 50%    | 1         |

### 三盘镜像（3-Way Mirror）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 |
|--------|----------|-----------|----------|--------|-----------|
| 3      | 1TB      | 3TB       | 1TB      | 33%    | 2         |
| 3      | 2TB      | 6TB       | 2TB      | 33%    | 2         |
| 3      | 4TB      | 12TB      | 4TB      | 33%    | 2         |
| 3      | 6TB      | 18TB      | 6TB      | 33%    | 2         |
| 3      | 8TB      | 24TB      | 8TB      | 33%    | 2         |
| 3      | 10TB     | 30TB      | 10TB     | 33%    | 2         |
| 3      | 12TB     | 36TB      | 12TB     | 33%    | 2         |
| 3      | 16TB     | 48TB      | 16TB     | 33%    | 2         |
| 3      | 20TB     | 60TB      | 20TB     | 33%    | 2         |

### 多组镜像配置

| 配置描述 | 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 |
|---------|--------|----------|-----------|----------|--------|-----------|
| 2×2盘镜像 | 4 | 4TB | 16TB | 8TB | 50% | 2 (每组1个) |
| 3×2盘镜像 | 6 | 4TB | 24TB | 12TB | 50% | 3 (每组1个) |
| 4×2盘镜像 | 8 | 4TB | 32TB | 16TB | 50% | 4 (每组1个) |
| 2×3盘镜像 | 6 | 4TB | 24TB | 8TB | 33% | 4 (每组2个) |
| 3×3盘镜像 | 9 | 4TB | 36TB | 12TB | 33% | 6 (每组2个) |
| 2×2盘镜像 | 4 | 8TB | 32TB | 16TB | 50% | 2 (每组1个) |
| 3×2盘镜像 | 6 | 8TB | 48TB | 24TB | 50% | 3 (每组1个) |

---

## RAID-Z1 容量表

### 单奇偶校验（RAID-5 类似）

#### 3-5 盘配置（推荐用于 RAID-Z1）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 最小IOPS比例 |
|--------|----------|-----------|----------|--------|-----------|-------------|
| 3      | 1TB      | 3TB       | 2TB      | 67%    | 1         | 33%         |
| 3      | 2TB      | 6TB       | 4TB      | 67%    | 1         | 33%         |
| 3      | 4TB      | 12TB      | 8TB      | 67%    | 1         | 33%         |
| 3      | 6TB      | 18TB      | 12TB     | 67%    | 1         | 33%         |
| 3      | 8TB      | 24TB      | 16TB     | 67%    | 1         | 33%         |
| 3      | 10TB     | 30TB      | 20TB     | 67%    | 1         | 33%         |
| 3      | 12TB     | 36TB      | 24TB     | 67%    | 1         | 33%         |
| | | | | | | |
| 4      | 1TB      | 4TB       | 3TB      | 75%    | 1         | 25%         |
| 4      | 2TB      | 8TB       | 6TB      | 75%    | 1         | 25%         |
| 4      | 4TB      | 16TB      | 12TB     | 75%    | 1         | 25%         |
| 4      | 6TB      | 24TB      | 18TB     | 75%    | 1         | 25%         |
| 4      | 8TB      | 32TB      | 24TB     | 75%    | 1         | 25%         |
| 4      | 10TB     | 40TB      | 30TB     | 75%    | 1         | 25%         |
| 4      | 12TB     | 48TB      | 36TB     | 75%    | 1         | 25%         |
| | | | | | | |
| 5      | 1TB      | 5TB       | 4TB      | 80%    | 1         | 20%         |
| 5      | 2TB      | 10TB      | 8TB      | 80%    | 1         | 20%         |
| 5      | 4TB      | 20TB      | 16TB     | 80%    | 1         | 20%         |
| 5      | 6TB      | 30TB      | 24TB     | 80%    | 1         | 20%         |
| 5      | 8TB      | 40TB      | 32TB     | 80%    | 1         | 20%         |
| 5      | 10TB     | 50TB      | 40TB     | 80%    | 1         | 20%         |
| 5      | 12TB     | 60TB      | 48TB     | 80%    | 1         | 20%         |

#### 6-9 盘配置（不推荐用于 RAID-Z1，建议用 RAID-Z2）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 备注 |
|--------|----------|-----------|----------|--------|-----------|------|
| 6      | 4TB      | 24TB      | 20TB     | 83%    | 1         | ⚠️ 推荐Z2 |
| 7      | 4TB      | 28TB      | 24TB     | 86%    | 1         | ⚠️ 推荐Z2 |
| 8      | 4TB      | 32TB      | 28TB     | 88%    | 1         | ⚠️ 推荐Z2 |
| 9      | 4TB      | 36TB      | 32TB     | 89%    | 1         | ⚠️ 推荐Z2 |
| 6      | 8TB      | 48TB      | 40TB     | 83%    | 1         | ⚠️ 推荐Z2 |
| 7      | 8TB      | 56TB      | 48TB     | 86%    | 1         | ⚠️ 推荐Z2 |
| 8      | 8TB      | 64TB      | 56TB     | 88%    | 1         | ⚠️ 推荐Z2 |
| 9      | 8TB      | 72TB      | 64TB     | 89%    | 1         | ⚠️ 推荐Z2 |

---

## RAID-Z2 容量表

### 双奇偶校验（RAID-6 类似）- 推荐配置

#### 4-6 盘配置

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 最小IOPS比例 |
|--------|----------|-----------|----------|--------|-----------|-------------|
| 4      | 1TB      | 4TB       | 2TB      | 50%    | 2         | 25%         |
| 4      | 2TB      | 8TB       | 4TB      | 50%    | 2         | 25%         |
| 4      | 4TB      | 16TB      | 8TB      | 50%    | 2         | 25%         |
| 4      | 6TB      | 24TB      | 12TB     | 50%    | 2         | 25%         |
| 4      | 8TB      | 32TB      | 16TB     | 50%    | 2         | 25%         |
| 4      | 10TB     | 40TB      | 20TB     | 50%    | 2         | 25%         |
| 4      | 12TB     | 48TB      | 24TB     | 50%    | 2         | 25%         |
| 4      | 16TB     | 64TB      | 32TB     | 50%    | 2         | 25%         |
| | | | | | | |
| 5      | 1TB      | 5TB       | 3TB      | 60%    | 2         | 20%         |
| 5      | 2TB      | 10TB      | 6TB      | 60%    | 2         | 20%         |
| 5      | 4TB      | 20TB      | 12TB     | 60%    | 2         | 20%         |
| 5      | 6TB      | 30TB      | 18TB     | 60%    | 2         | 20%         |
| 5      | 8TB      | 40TB      | 24TB     | 60%    | 2         | 20%         |
| 5      | 10TB     | 50TB      | 30TB     | 60%    | 2         | 20%         |
| 5      | 12TB     | 60TB      | 36TB     | 60%    | 2         | 20%         |
| 5      | 16TB     | 80TB      | 48TB     | 60%    | 2         | 20%         |
| | | | | | | |
| 6      | 1TB      | 6TB       | 4TB      | 67%    | 2         | 17%         |
| 6      | 2TB      | 12TB      | 8TB      | 67%    | 2         | 17%         |
| 6      | 4TB      | 24TB      | 16TB     | 67%    | 2         | 17%         |
| 6      | 6TB      | 36TB      | 24TB     | 67%    | 2         | 17%         |
| 6      | 8TB      | 48TB      | 32TB     | 67%    | 2         | 17%         |
| 6      | 10TB     | 60TB      | 40TB     | 67%    | 2         | 17%         |
| 6      | 12TB     | 72TB      | 48TB     | 67%    | 2         | 17%         |
| 6      | 16TB     | 96TB      | 64TB     | 67%    | 2         | 17%         |

#### 7-10 盘配置（推荐用于 RAID-Z2）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 最小IOPS比例 |
|--------|----------|-----------|----------|--------|-----------|-------------|
| 7      | 4TB      | 28TB      | 20TB     | 71%    | 2         | 14%         |
| 8      | 4TB      | 32TB      | 24TB     | 75%    | 2         | 13%         |
| 9      | 4TB      | 36TB      | 28TB     | 78%    | 2         | 11%         |
| 10     | 4TB      | 40TB      | 32TB     | 80%    | 2         | 10%         |
| | | | | | | |
| 7      | 8TB      | 56TB      | 40TB     | 71%    | 2         | 14%         |
| 8      | 8TB      | 64TB      | 48TB     | 75%    | 2         | 13%         |
| 9      | 8TB      | 72TB      | 56TB     | 78%    | 2         | 11%         |
| 10     | 8TB      | 80TB      | 64TB     | 80%    | 2         | 10%         |
| | | | | | | |
| 7      | 12TB     | 84TB      | 60TB     | 71%    | 2         | 14%         |
| 8      | 12TB     | 96TB      | 72TB     | 75%    | 2         | 13%         |
| 9      | 12TB     | 108TB     | 84TB     | 78%    | 2         | 11%         |
| 10     | 12TB     | 120TB     | 96TB     | 80%    | 2         | 10%         |
| | | | | | | |
| 7      | 16TB     | 112TB     | 80TB     | 71%    | 2         | 14%         |
| 8      | 16TB     | 128TB     | 96TB     | 75%    | 2         | 13%         |
| 9      | 16TB     | 144TB     | 112TB    | 78%    | 2         | 11%         |
| 10     | 16TB     | 160TB     | 128TB    | 80%    | 2         | 10%         |

#### 11-15 盘配置（大型阵列，考虑 Z3 或 dRAID）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 备注 |
|--------|----------|-----------|----------|--------|-----------|------|
| 11     | 8TB      | 88TB      | 72TB     | 82%    | 2         | 可考虑Z3 |
| 12     | 8TB      | 96TB      | 80TB     | 83%    | 2         | 可考虑Z3 |
| 13     | 8TB      | 104TB     | 88TB     | 85%    | 2         | 推荐Z3 |
| 14     | 8TB      | 112TB     | 96TB     | 86%    | 2         | 推荐Z3 |
| 15     | 8TB      | 120TB     | 104TB    | 87%    | 2         | 推荐Z3 |
| | | | | | | |
| 11     | 12TB     | 132TB     | 108TB    | 82%    | 2         | 可考虑Z3 |
| 12     | 12TB     | 144TB     | 120TB    | 83%    | 2         | 可考虑Z3 |
| 13     | 12TB     | 156TB     | 132TB    | 85%    | 2         | 推荐Z3 |
| 14     | 12TB     | 168TB     | 144TB    | 86%    | 2         | 推荐Z3 |
| 15     | 12TB     | 180TB     | 156TB    | 87%    | 2         | 推荐Z3 |

---

## RAID-Z3 容量表

### 三重奇偶校验（高可靠性）

#### 5-8 盘配置（最小推荐）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 | 最小IOPS比例 |
|--------|----------|-----------|----------|--------|-----------|-------------|
| 5      | 4TB      | 20TB      | 8TB      | 40%    | 3         | 20%         |
| 5      | 8TB      | 40TB      | 16TB     | 40%    | 3         | 20%         |
| 5      | 12TB     | 60TB      | 24TB     | 40%    | 3         | 20%         |
| 5      | 16TB     | 80TB      | 32TB     | 40%    | 3         | 20%         |
| | | | | | | |
| 6      | 4TB      | 24TB      | 12TB     | 50%    | 3         | 17%         |
| 6      | 8TB      | 48TB      | 24TB     | 50%    | 3         | 17%         |
| 6      | 12TB     | 72TB      | 36TB     | 50%    | 3         | 17%         |
| 6      | 16TB     | 96TB      | 48TB     | 50%    | 3         | 17%         |
| | | | | | | |
| 7      | 4TB      | 28TB      | 16TB     | 57%    | 3         | 14%         |
| 7      | 8TB      | 56TB      | 32TB     | 57%    | 3         | 14%         |
| 7      | 12TB     | 84TB      | 48TB     | 57%    | 3         | 14%         |
| 7      | 16TB     | 112TB     | 64TB     | 57%    | 3         | 14%         |
| | | | | | | |
| 8      | 4TB      | 32TB      | 20TB     | 63%    | 3         | 13%         |
| 8      | 8TB      | 64TB      | 40TB     | 63%    | 3         | 13%         |
| 8      | 12TB     | 96TB      | 60TB     | 63%    | 3         | 13%         |
| 8      | 16TB     | 128TB     | 80TB     | 63%    | 3         | 13%         |

#### 9-15 盘配置（推荐用于 RAID-Z3）

| 磁盘数 | 单盘容量 | 原始总容量 | 可用容量 | 可用率 | 允许故障数 |
|--------|----------|-----------|----------|--------|-----------|
| 9      | 8TB      | 72TB      | 48TB     | 67%    | 3         |
| 10     | 8TB      | 80TB      | 56TB     | 70%    | 3         |
| 11     | 8TB      | 88TB      | 64TB     | 73%    | 3         |
| 12     | 8TB      | 96TB      | 72TB     | 75%    | 3         |
| 13     | 8TB      | 104TB     | 80TB     | 77%    | 3         |
| 14     | 8TB      | 112TB     | 88TB     | 79%    | 3         |
| 15     | 8TB      | 120TB     | 96TB     | 80%    | 3         |
| | | | | | |
| 9      | 12TB     | 108TB     | 72TB     | 67%    | 3         |
| 10     | 12TB     | 120TB     | 84TB     | 70%    | 3         |
| 11     | 12TB     | 132TB     | 96TB     | 73%    | 3         |
| 12     | 12TB     | 144TB     | 108TB    | 75%    | 3         |
| 13     | 12TB     | 156TB     | 120TB    | 77%    | 3         |
| 14     | 12TB     | 168TB     | 132TB    | 79%    | 3         |
| 15     | 12TB     | 180TB     | 144TB    | 80%    | 3         |
| | | | | | |
| 9      | 16TB     | 144TB     | 96TB     | 67%    | 3         |
| 10     | 16TB     | 160TB     | 112TB    | 70%    | 3         |
| 11     | 16TB     | 176TB     | 128TB    | 73%    | 3         |
| 12     | 16TB     | 192TB     | 144TB    | 75%    | 3         |
| 13     | 16TB     | 208TB     | 160TB    | 77%    | 3         |
| 14     | 16TB     | 224TB     | 176TB    | 79%    | 3         |
| 15     | 16TB     | 240TB     | 192TB    | 80%    | 3         |

---

## dRAID 容量表

### dRAID1（单奇偶，带热备）

默认配置：D=8（数据盘），无热备

| 磁盘数 | 单盘容量 | 配置 | 可用容量 | 可用率 | 允许故障数 | 备注 |
|--------|----------|------|----------|--------|-----------|------|
| 8      | 8TB      | draid1:8d:0s | 56TB | 88% | 1 | 无热备 |
| 9      | 8TB      | draid1:8d:1s | 56TB | 78% | 1+1热备 | 1个热备 |
| 10     | 8TB      | draid1:8d:2s | 56TB | 70% | 1+2热备 | 2个热备 |
| 12     | 8TB      | draid1:8d:0s | 88TB | 92% | 1 | 无热备 |
| 16     | 8TB      | draid1:8d:0s | 112TB | 88% | 1 | 无热备 |
| 16     | 8TB      | draid1:8d:2s | 98TB | 77% | 1+2热备 | 2个热备 |

带热备配置示例（8TB 磁盘）：

| 磁盘数 | D:S配置 | 原始总容量 | 可用容量 | 可用率 | 热备数 | 允许故障 |
|--------|---------|-----------|----------|--------|--------|---------|
| 10     | 8d:1s   | 80TB      | 63TB     | 79%    | 1      | 1+1热备 |
| 11     | 8d:2s   | 88TB      | 63TB     | 72%    | 2      | 1+2热备 |
| 12     | 8d:1s   | 96TB      | 77TB     | 80%    | 1      | 1+1热备 |
| 15     | 8d:2s   | 120TB     | 91TB     | 76%    | 2      | 1+2热备 |

### dRAID2（双奇偶，带热备）

默认配置：D=8（数据盘）

| 磁盘数 | 单盘容量 | 配置 | 可用容量 | 可用率 | 允许故障数 | 备注 |
|--------|----------|------|----------|--------|-----------|------|
| 10     | 8TB      | draid2:8d:0s | 64TB | 80% | 2 | 无热备 |
| 11     | 8TB      | draid2:8d:1s | 64TB | 73% | 2+1热备 | 1个热备 |
| 12     | 8TB      | draid2:8d:2s | 64TB | 67% | 2+2热备 | 2个热备 |
| 15     | 8TB      | draid2:8d:1s | 88TB | 73% | 2+1热备 | 1个热备 |
| 18     | 8TB      | draid2:8d:2s | 112TB | 70% | 2+2热备 | 2个热备 |

带热备配置示例（12TB 磁盘）：

| 磁盘数 | D:S配置 | 原始总容量 | 可用容量 | 可用率 | 热备数 | 允许故障 |
|--------|---------|-----------|----------|--------|--------|---------|
| 10     | 8d:0s   | 120TB     | 96TB     | 80%    | 0      | 2       |
| 11     | 8d:1s   | 132TB     | 96TB     | 73%    | 1      | 2+1热备 |
| 12     | 8d:2s   | 144TB     | 96TB     | 67%    | 2      | 2+2热备 |
| 15     | 8d:1s   | 180TB     | 132TB    | 73%    | 1      | 2+1热备 |
| 20     | 8d:2s   | 240TB     | 168TB    | 70%    | 2      | 2+2热备 |

### dRAID3（三重奇偶，带热备）

| 磁盘数 | 单盘容量 | 配置 | 可用容量 | 可用率 | 允许故障数 |
|--------|----------|------|----------|--------|-----------|
| 12     | 12TB     | draid3:8d:0s | 108TB | 75% | 3 |
| 13     | 12TB     | draid3:8d:1s | 108TB | 69% | 3+1热备 |
| 15     | 12TB     | draid3:8d:1s | 132TB | 73% | 3+1热备 |
| 18     | 12TB     | draid3:8d:2s | 156TB | 72% | 3+2热备 |
| 20     | 12TB     | draid3:8d:2s | 180TB | 75% | 3+2热备 |

---

## 常见磁盘容量对照

### 实际容量 vs 标称容量

制造商使用十进制（1TB = 1000GB），而操作系统使用二进制（1TiB = 1024GiB）。

| 标称容量 | 十进制字节 | 二进制（GiB） | 二进制（TiB） | 实际可用约 |
|----------|-----------|-------------|-------------|-----------|
| 1TB      | 1,000,000,000,000 | 931 GiB | 0.909 TiB | 900 GB |
| 2TB      | 2,000,000,000,000 | 1,863 GiB | 1.819 TiB | 1.8 TB |
| 3TB      | 3,000,000,000,000 | 2,794 GiB | 2.728 TiB | 2.7 TB |
| 4TB      | 4,000,000,000,000 | 3,725 GiB | 3.638 TiB | 3.6 TB |
| 6TB      | 6,000,000,000,000 | 5,588 GiB | 5.456 TiB | 5.4 TB |
| 8TB      | 8,000,000,000,000 | 7,451 GiB | 7.276 TiB | 7.2 TB |
| 10TB     | 10,000,000,000,000 | 9,313 GiB | 9.095 TiB | 9.0 TB |
| 12TB     | 12,000,000,000,000 | 11,176 GiB | 10.914 TiB | 10.8 TB |
| 14TB     | 14,000,000,000,000 | 13,039 GiB | 12.734 TiB | 12.6 TB |
| 16TB     | 16,000,000,000,000 | 14,901 GiB | 14.552 TiB | 14.4 TB |
| 18TB     | 18,000,000,000,000 | 16,764 GiB | 16.370 TiB | 16.2 TB |
| 20TB     | 20,000,000,000,000 | 18,626 GiB | 18.190 TiB | 18.0 TB |

### 常见配置实际容量示例

**6×8TB RAID-Z2 实际容量**：
- 理论：(6-2) × 8TB = 32TB
- 实际：(6-2) × 7.2TB ≈ 28.8TB
- ZFS 开销后：≈ 28TB

**8×12TB RAID-Z2 实际容量**：
- 理论：(8-2) × 12TB = 72TB
- 实际：(8-2) × 10.8TB ≈ 64.8TB
- ZFS 开销后：≈ 63TB

---

## 容量计算器

### Bash 计算脚本

```bash
#!/bin/bash
# 文件名: zfs-capacity-calculator.sh
# 用途: 计算 ZFS RAID 配置的可用容量

# 使用方法
show_usage() {
    cat <<EOF
ZFS 容量计算器

用法: $0 <raid_type> <disk_count> <disk_size_tb>

RAID 类型:
  mirror     - 镜像
  raidz1     - 单奇偶校验
  raidz2     - 双奇偶校验
  raidz3     - 三重奇偶校验
  draid1     - 分布式单奇偶（带1个热备）
  draid2     - 分布式双奇偶（带1个热备）

示例:
  $0 raidz2 6 8    # 6个8TB磁盘的RAID-Z2
  $0 mirror 2 4    # 2个4TB磁盘的镜像
  $0 draid2 12 10  # 12个10TB磁盘的dRAID2
EOF
}

# 参数检查
if [ $# -ne 3 ]; then
    show_usage
    exit 1
fi

RAID_TYPE=$1
DISK_COUNT=$2
DISK_SIZE=$3

# 验证输入
if ! [[ "$DISK_COUNT" =~ ^[0-9]+$ ]] || [ "$DISK_COUNT" -lt 2 ]; then
    echo "错误: 磁盘数量必须是大于1的整数"
    exit 1
fi

if ! [[ "$DISK_SIZE" =~ ^[0-9]+$ ]] || [ "$DISK_SIZE" -lt 1 ]; then
    echo "错误: 磁盘大小必须是正整数"
    exit 1
fi

# 计算容量
calculate_capacity() {
    local raid=$1
    local count=$2
    local size=$3
    local parity=0
    local spare=0
    local data_ratio=1.0

    case $raid in
        mirror)
            parity=1
            usable=$((size))
            ratio="1/$count"
            fault_tolerance=1
            ;;
        raidz1|raidz)
            parity=1
            usable=$(( (count - parity) * size ))
            ratio="$(($count-1))/$count"
            fault_tolerance=1
            ;;
        raidz2)
            parity=2
            usable=$(( (count - parity) * size ))
            ratio="$(($count-2))/$count"
            fault_tolerance=2
            ;;
        raidz3)
            parity=3
            usable=$(( (count - parity) * size ))
            ratio="$(($count-3))/$count"
            fault_tolerance=3
            ;;
        draid1)
            spare=1
            parity=1
            # 简化计算: (N-S) * 0.875 (假设D=8)
            usable=$(( (count - spare) * size * 875 / 1000 ))
            ratio="≈87.5%"
            fault_tolerance="1+${spare}热备"
            ;;
        draid2)
            spare=1
            parity=2
            # 简化计算: (N-S) * 0.8 (假设D=8)
            usable=$(( (count - spare) * size * 800 / 1000 ))
            ratio="≈80%"
            fault_tolerance="2+${spare}热备"
            ;;
        *)
            echo "错误: 不支持的 RAID 类型: $raid"
            echo "支持的类型: mirror, raidz1, raidz2, raidz3, draid1, draid2"
            exit 1
            ;;
    esac

    # 计算实际容量（考虑进制差异）
    actual_disk=$(echo "$size * 0.909" | bc)
    actual_usable=$(echo "$usable * 0.909" | bc)

    # ZFS 开销（约2%）
    final_usable=$(echo "$actual_usable * 0.98" | bc)

    # 输出结果
    cat <<EOF

========================================
ZFS 容量计算结果
========================================

配置信息:
  RAID 类型: $raid
  磁盘数量: $count
  单盘大小: ${size}TB (标称)

理论计算:
  原始总容量: $(($count * $size))TB
  可用容量: ${usable}TB
  可用率: $ratio

实际容量（考虑进制差异）:
  单盘实际: ${actual_disk}TB
  可用实际: ${actual_usable}TB
  最终可用: ${final_usable}TB (扣除ZFS开销)

冗余信息:
  允许故障: $fault_tolerance
  冗余开销: $(( $count * $size - $usable ))TB

========================================
EOF

    # 推荐建议
    echo "配置建议:"
    case $raid in
        mirror)
            if [ $count -eq 2 ]; then
                echo "  ✓ 2盘镜像是标准配置"
            else
                echo "  ⚠ 建议使用2盘镜像，或考虑RAID-Z"
            fi
            ;;
        raidz1)
            if [ $count -ge 3 ] && [ $count -le 5 ]; then
                echo "  ✓ RAID-Z1适合3-5个磁盘"
            else
                echo "  ⚠ 建议使用RAID-Z2以提高可靠性"
            fi
            ;;
        raidz2)
            if [ $count -ge 4 ] && [ $count -le 10 ]; then
                echo "  ✓ RAID-Z2是最推荐的配置"
            elif [ $count -gt 10 ]; then
                echo "  ⚠ 磁盘数较多，可考虑RAID-Z3或dRAID"
            fi
            ;;
        raidz3)
            if [ $count -ge 5 ]; then
                echo "  ✓ RAID-Z3提供最高冗余"
            fi
            ;;
    esac
}

# 执行计算
calculate_capacity "$RAID_TYPE" "$DISK_COUNT" "$DISK_SIZE"
```

### 使用示例

```bash
# 保存脚本
chmod +x zfs-capacity-calculator.sh

# 计算6×8TB RAID-Z2
./zfs-capacity-calculator.sh raidz2 6 8

# 计算2×4TB 镜像
./zfs-capacity-calculator.sh mirror 2 4

# 计算12×10TB dRAID2
./zfs-capacity-calculator.sh draid2 12 10
```

### Python 计算器（Web版）

```python
#!/usr/bin/env python3
"""
ZFS 容量计算器（Web版）
"""

def calculate_zfs_capacity(raid_type, disk_count, disk_size_tb):
    """
    计算 ZFS 可用容量

    参数:
        raid_type: RAID类型 (mirror, raidz1, raidz2, raidz3)
        disk_count: 磁盘数量
        disk_size_tb: 单盘容量(TB)

    返回:
        dict: 包含各种容量信息
    """

    # 基础计算
    raw_capacity = disk_count * disk_size_tb

    if raid_type == 'mirror':
        usable = disk_size_tb
        parity = disk_count - 1
        fault_tolerance = 1
        ratio = 1 / disk_count
    elif raid_type == 'raidz1':
        usable = (disk_count - 1) * disk_size_tb
        parity = 1
        fault_tolerance = 1
        ratio = (disk_count - 1) / disk_count
    elif raid_type == 'raidz2':
        usable = (disk_count - 2) * disk_size_tb
        parity = 2
        fault_tolerance = 2
        ratio = (disk_count - 2) / disk_count
    elif raid_type == 'raidz3':
        usable = (disk_count - 3) * disk_size_tb
        parity = 3
        fault_tolerance = 3
        ratio = (disk_count - 3) / disk_count
    else:
        raise ValueError(f"不支持的RAID类型: {raid_type}")

    # 实际容量（1TB = 0.909TiB）
    actual_usable = usable * 0.909

    # ZFS开销（2%）
    final_usable = actual_usable * 0.98

    return {
        'raw_capacity': raw_capacity,
        'usable_capacity': usable,
        'actual_usable': round(actual_usable, 2),
        'final_usable': round(final_usable, 2),
        'ratio': round(ratio * 100, 1),
        'parity_overhead': raw_capacity - usable,
        'fault_tolerance': fault_tolerance
    }


def print_capacity_table(disk_sizes, disk_counts, raid_type):
    """打印容量对照表"""
    print(f"\n{raid_type.upper()} 容量对照表\n")
    print("磁盘数 | ", end="")
    for size in disk_sizes:
        print(f"{size}TB      | ", end="")
    print("\n" + "-" * 80)

    for count in disk_counts:
        print(f"{count:^6} | ", end="")
        for size in disk_sizes:
            result = calculate_zfs_capacity(raid_type, count, size)
            print(f"{result['final_usable']:6.1f}TB | ", end="")
        print()


if __name__ == "__main__":
    # 示例：生成RAID-Z2表格
    disk_sizes = [4, 6, 8, 10, 12]
    disk_counts = [4, 5, 6, 7, 8, 9, 10]

    print_capacity_table(disk_sizes, disk_counts, 'raidz2')

    # 单个计算示例
    print("\n单个配置计算示例：")
    result = calculate_zfs_capacity('raidz2', 6, 8)
    print(f"6×8TB RAID-Z2:")
    print(f"  原始容量: {result['raw_capacity']}TB")
    print(f"  理论可用: {result['usable_capacity']}TB")
    print(f"  实际可用: {result['final_usable']}TB")
    print(f"  可用率: {result['ratio']}%")
    print(f"  允许故障: {result['fault_tolerance']}个磁盘")
```

---

## 配置推荐

### 按用途推荐

#### 1. 家庭/小型办公室（2-6 盘）

| 需求 | 推荐配置 | 磁盘数 | 示例 | 可用率 |
|------|---------|--------|------|--------|
| 基础存储 | 双盘镜像 | 2 | 2×4TB | 50% |
| 平衡性能容量 | RAID-Z1 | 3-5 | 4×4TB | 75% |
| 高可靠性 | RAID-Z2 | 4-6 | 6×4TB | 67% |

**推荐**：
- **预算有限**：3×4TB RAID-Z1（8TB可用）
- **平衡选择**：4×4TB RAID-Z2（8TB可用，允许2盘故障）
- **高性能**：4×2TB 镜像（4TB可用）

#### 2. 中小企业（6-12 盘）

| 需求 | 推荐配置 | 磁盘数 | 示例 | 可用率 |
|------|---------|--------|------|--------|
| 标准存储 | RAID-Z2 | 6-10 | 8×8TB | 75% |
| 高可靠性 | RAID-Z3 | 8-12 | 10×8TB | 70% |
| 高性能 | 多组镜像 | 6-12 | 6×(2×4TB) | 50% |

**推荐**：
- **标准配置**：8×8TB RAID-Z2（48TB可用）
- **高可靠性**：10×8TB RAID-Z3（56TB可用）
- **高性能**：4×(2×6TB) 镜像（24TB可用）

#### 3. 大型企业/数据中心（12+ 盘）

| 需求 | 推荐配置 | 磁盘数 | 示例 | 可用率 |
|------|---------|--------|------|--------|
| 大容量 | RAID-Z2 | 10-15 | 12×12TB | 83% |
| 极高可靠 | RAID-Z3 | 12-15 | 15×12TB | 80% |
| 快速恢复 | dRAID2 | 12+ | 16×12TB | 73% |

**推荐**：
- **标准大型**：12×12TB RAID-Z2（120TB可用）
- **关键数据**：15×12TB RAID-Z3（144TB可用）
- **快速恢复**：16×12TB dRAID2（140TB可用，带热备）

### 按性能需求推荐

#### 高 IOPS 需求（数据库、虚拟机）

1. **镜像**（最高IOPS）
   - 配置：多组2盘镜像
   - 示例：6×(2×960GB SSD)
   - IOPS：≈100%单盘性能

2. **RAID-Z1 + SSD**
   - 配置：4-5个SSD
   - 示例：4×1TB NVMe
   - IOPS：≈80%

#### 平衡性能/容量

1. **RAID-Z2**（推荐）
   - 配置：6-10个磁盘
   - 示例：8×8TB
   - 性能：良好

2. **混合池**
   - 数据：8×8TB RAID-Z2（HDD）
   - 日志：2×500GB NVMe镜像（SLOG）
   - 缓存：2×2TB SSD（L2ARC）

#### 高容量需求

1. **RAID-Z2**
   - 配置：10-15个大容量盘
   - 示例：12×16TB
   - 可用：160TB

2. **RAID-Z3**
   - 配置：12-15个大容量盘
   - 示例：15×16TB
   - 可用：192TB

### 按预算推荐

#### 预算型（<$1000）

| 配置 | 磁盘 | 成本估算 | 可用容量 | 备注 |
|------|------|---------|---------|------|
| 3×4TB Z1 | 3×$80 | $240 | 8TB | 基础配置 |
| 4×4TB Z2 | 4×$80 | $320 | 8TB | 更可靠 |
| 2×8TB Mirror | 2×$160 | $320 | 8TB | 高性能 |

#### 中端型（$1000-$3000）

| 配置 | 磁盘 | 成本估算 | 可用容量 | 备注 |
|------|------|---------|---------|------|
| 6×8TB Z2 | 6×$160 | $960 | 32TB | 推荐 |
| 8×6TB Z2 | 8×$120 | $960 | 36TB | 好平衡 |
| 5×12TB Z2 | 5×$240 | $1200 | 36TB | 大盘 |

#### 高端型（>$3000）

| 配置 | 磁盘 | 成本估算 | 可用容量 | 备注 |
|------|------|---------|---------|------|
| 10×12TB Z2 | 10×$240 | $2400 | 96TB | 大容量 |
| 12×12TB Z3 | 12×$240 | $2880 | 108TB | 高可靠 |
| 8×16TB Z2 | 8×$320 | $2560 | 96TB | 最新大盘 |

### 不推荐的配置

❌ **避免以下配置**：

1. **单盘池**
   - 无冗余，数据风险高
   - 仅用于临时/非重要数据

2. **>9 盘 RAID-Z1**
   - Resilver 时间过长
   - 单盘故障风险高
   - 改用 RAID-Z2

3. **<4 盘 RAID-Z2**
   - 可用率太低（50%）
   - 不如镜像

4. **异构磁盘**
   - 不同大小磁盘混用
   - 浪费容量
   - 性能不一致

5. **过多磁盘的单个 vdev**
   - >15 盘 RAID-Z2/Z3
   - Resilver 时间极长
   - 考虑拆分或使用 dRAID

---

## 快速参考卡

### 最常用配置速查

```
家庭用户推荐：
┌──────────────────────────────────────┐
│ 4×4TB RAID-Z2 = 8TB 可用             │
│ 成本：~$320                          │
│ 允许 2 盘故障                         │
└──────────────────────────────────────┘

小型企业推荐：
┌──────────────────────────────────────┐
│ 8×8TB RAID-Z2 = 48TB 可用            │
│ 成本：~$1280                         │
│ 允许 2 盘故障                         │
└──────────────────────────────────────┘

大型企业推荐：
┌──────────────────────────────────────┐
│ 12×12TB RAID-Z2 = 120TB 可用         │
│ 成本：~$2880                         │
│ 允许 2 盘故障                         │
└──────────────────────────────────────┘
```

### 容量效率对比

```
配置类型     可用率    允许故障    IOPS   推荐场景
─────────────────────────────────────────────────
镜像(2盘)    50%      1个         高     高性能
RAID-Z1      67-80%   1个         中     家庭/小型
RAID-Z2      50-80%   2个         中     通用推荐⭐
RAID-Z3      40-80%   3个         低     关键数据
dRAID2       70-80%   2+热备      中     大型阵列
```

### 磁盘数量建议

```
磁盘数    推荐RAID        可用率    备注
─────────────────────────────────────────────
2         镜像            50%       标准配置
3         RAID-Z1         67%       入门级
4-5       RAID-Z1/Z2      60-75%    家庭推荐
6-10      RAID-Z2         67-80%    企业推荐⭐
8-15      RAID-Z3         63-80%    高可靠
12+       dRAID2          70-80%    大型系统
```

---

## 附录：完整命令参考

### 创建各种 RAID 配置

```bash
# 镜像
zpool create mypool mirror /dev/sda /dev/sdb

# RAID-Z1
zpool create mypool raidz /dev/sd{a,b,c,d}

# RAID-Z2（推荐）
zpool create mypool raidz2 /dev/sd{a,b,c,d,e,f}

# RAID-Z3
zpool create mypool raidz3 /dev/sd{a,b,c,d,e,f,g,h}

# dRAID2（8个数据盘，1个热备）
zpool create mypool draid2:8d:1s /dev/sd{a..l}

# 多组镜像
zpool create mypool \
    mirror /dev/sda /dev/sdb \
    mirror /dev/sdc /dev/sdd \
    mirror /dev/sde /dev/sdf

# 混合配置
zpool create mypool \
    raidz2 /dev/sd{a,b,c,d,e,f} \
    log mirror /dev/nvme0n1 /dev/nvme1n1 \
    cache /dev/ssd1 /dev/ssd2
```

### 查看容量信息

```bash
# 查看池容量
zpool list -o name,size,allocated,free,capacity,health

# 查看碎片率
zpool list -o name,size,allocated,free,fragmentation

# 查看详细空间使用
zfs list -o space poolname

# 查看压缩比
zfs get compressratio poolname

# 查看去重比
zfs get dedupratio poolname
```

---

**文档版本**: 1.0
**创建日期**: 2025-01-13
**适用版本**: OpenZFS 2.0+
**许可**: CDDL-1.0

---

## 结语

选择合适的 ZFS RAID 配置需要平衡：
- **容量**：可用空间百分比
- **性能**：IOPS 和吞吐量
- **可靠性**：允许故障数
- **成本**：磁盘数量和价格

**通用建议**：
- **家庭用户**：4-6 盘 RAID-Z2
- **小型企业**：6-10 盘 RAID-Z2
- **大型企业**：10-15 盘 RAID-Z2/Z3 或 dRAID2
- **关键数据**：RAID-Z3 或带热备的 dRAID

使用本文档中的计算器可以帮助你为特定需求选择最佳配置。
