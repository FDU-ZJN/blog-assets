---
title: BOOM源码与《超标量处理器设计》学习
categories: 处理器
tags:
  - IC
cover: /img/cover_11.jpg
highlight_shrink: true
abbrlink: 3740229498
date: 2025-06-18 18:39:15
---

# BOOM

## 寄存器重命名

1. **RenameStage** - 重命名阶段顶层模块
2. **RenameMapTable** - 寄存器映射表
3. **RenameFreeList** - 空闲物理寄存器列表
4. **RenameBusyTable** - 寄存器忙状态表

### RenameStage

#### 整体两阶段流水线：

-  阶段1(Ren1)：接收来自解码阶段的微操作(uop)
-  阶段2(Ren2)：完成重命名并发送到调度队列

-  通过`MapTable`完成逻辑寄存器到物理寄存器的映射
-  通过`FreeList`分配新的物理寄存器
-  通过`BusyTable`跟踪寄存器忙状态
-  处理分支预测错误时的恢复

#### AbstractRenameStage——寄存器重命名的**基础抽象类**

micro微操作集定义在micro-op.scala

包含了**指令基本信息**，**逻辑寄存器编号**，**物理寄存器编号（重命名阶段分配）**，**寄存器忙状态（`BusyTable` 提供）**，**分支预测相关**，**执行信息**

##### 1. **流水线控制信号**

| 信号         | 方向   | 描述                                                         |
| :----------- | :----- | :----------------------------------------------------------- |
| `ren_stalls` | Output | 每个指令槽的重命名阻塞信号（如空闲寄存器不足时阻塞对应指令） |
| `kill`       | Input  | 全局流水线清除信号（分支预测失败或异常时触发）               |
| `dec_fire`   | Input  | 来自解码阶段的指令有效信号（标记哪些指令需要被处理）         |
| `dec_uops`   | Input  | 解码后的微操作（MicroOp）输入，包含逻辑寄存器编号等信息      |

##### 2. **重命名输出**

| 信号        | 方向   | 描述                                                         |
| :---------- | :----- | :----------------------------------------------------------- |
| `ren2_mask` | Output | 重命名后有效的指令掩码（标记哪些指令成功通过重命名阶段）     |
| `ren2_uops` | Output | 重命名完成后的微操作（已替换为物理寄存器编号，并更新忙状态） |

##### 3. **分支处理**

| 信号       | 方向  | 描述                                           |
| :--------- | :---- | :--------------------------------------------- |
| `brupdate` | Input | 分支更新信息（包含分支误预测和恢复所需的标签） |

##### 4. **分发接口**

| 信号        | 方向  | 描述                                                         |
| :---------- | :---- | :----------------------------------------------------------- |
| `dis_fire`  | Input | 后续分发阶段（Dispatch）的确认信号（标记哪些指令被成功接收） |
| `dis_ready` | Input | 分发阶段是否准备好接收新指令                                 |

##### 5. **写回唤醒**

| 信号      | 方向  | 描述                                                         |
| :-------- | :---- | :----------------------------------------------------------- |
| `wakeups` | Input | 来自执行单元的写回信号（用于更新忙状态表，标记寄存器已就绪） |

##### 6. **提交与回滚**

| 信号         | 方向  | 描述                                             |
| :----------- | :---- | :----------------------------------------------- |
| `com_valids` | Input | 提交阶段的有效信号（标记哪些指令已提交）         |
| `com_uops`   | Input | 提交的微操作（用于释放物理寄存器）               |
| `rbk_valids` | Input | 回滚指令的有效信号（分支误预测时需要撤销的指令） |
| `rollback`   | Input | 回滚触发信号（与`rbk_valids`配合使用）           |

##### 7. **调试接口**

| 信号              | 方向   | 描述                                                       |
| :---------------- | :----- | :--------------------------------------------------------- |
| `debug_rob_empty` | Input  | ROB（重排序缓冲区）是否为空的调试信号                      |
| `debug`           | Output | 重命名阶段的调试信息（包含空闲列表、待释放列表和忙状态表） |

#### 两级流水线寄存器逻辑

**ren1**接收来自解码阶段（Decode）的指令。

```scala
ren1_fire(w) := io.dec_fire(w)       // 指令是否有效
ren1_uops(w) := io.dec_uops(w)       // 微操作数据
```

**ren2**完成寄存器重命名，并等待分发阶段（Dispatch）接收。

```scala
ren2_fire := io.dis_fire             // 分发阶段确认接收
ren2_ready := io.dis_ready           // 分发阶段是否就绪
```

流水线寄存器 缓存 `ren1` 的数据，直到 `ren2` 可以接收。

```scala
when (io.kill) {
  r_valid := false.B                 // 清除流水线（分支预测失败）
} .elsewhen (ren2_ready) {
  r_valid := ren1_fire(w)            // 新指令进入 ren2
  next_uop := ren1_uops(w)           // 更新微操作
} .otherwise {
  r_valid := r_valid && !ren2_fire(w) // 如果指令未被接收，保持有效
  next_uop := r_uop                  // 保持当前微操作
}
```
- **`kill`**：全局清除信号（如分支预测失败）。
- **`ren2_ready`**：如果分发阶段就绪，`ren2` 可以接收新指令。
- **`ren2_fire`**：如果分发阶段接收了当前指令，则 `r_valid` 被清除。

---

**寄存器重命名 (`BypassAllocations` 调用)**

在更新 `r_uop` 时，调用 `BypassAllocations` 处理旁路：
```scala
r_uop := GetNewUopAndBrMask(BypassAllocations(next_uop, ren2_uops, ren2_alloc_reqs), io.brupdate)
```
`ren2_uops`：当前 `ren2` 阶段的所有指令（用于同一周期旁路）。

`ren2_alloc_reqs`：标记哪些指令修改了寄存器。

---

**输出 (`io.ren2_mask` 和 `io.ren2_uops`)**

**`io.ren2_mask`**：标记哪些 `ren2` 指令有效：

```scala
io.ren2_mask := ren2_valids
```

**`io.ren2_uops`**：输出重命名后的微操作（包含物理寄存器编号和忙状态）：

```scala
ren2_uops(w) := r_uop
```

#### 旁路处理

处理 **同一周期内指令间的寄存器依赖**（如 `WAW`/`WAR` 冲突），确保后续指令能读取最新的物理寄存器。

```scala
def BypassAllocations(uop: MicroOp, older_uops: Seq[MicroOp], alloc_reqs: Seq[Bool]): MicroOp
```

`uop`：当前指令的微操作。

`older_uops`：同一周期内更早处理的指令。

`alloc_reqs`：标记哪些指令修改了寄存器。

**检测依赖**：检查当前指令的源寄存器（`lrs1`/`lrs2`/`lrs3`）是否被前面的指令修改。

```scala
val bypass_hits_rs1 = (older_uops zip alloc_reqs).map { case (r,a) => a && r.ldst === uop.lrs1 }
```

**选择最新寄存器**：使用 `PriorityEncoderOH` 选择最先修改的物理寄存器。

```scala
when (do_bypass_rs1) { bypassed_uop.prs1 := Mux1H(bypass_sel_rs1, bypass_pdsts) }
```

**更新忙状态**：如果发生旁路，标记寄存器为“忙”。

```scala
bypassed_uop.prs1_busy := uop.prs1_busy || do_bypass_rs1
```

### **RenameMapTable**

**功能**：维护逻辑寄存器 → 物理寄存器的映射。

```scala
maptable.io.map_reqs   // 查询源寄存器（lrs1/lrs2/lrs3）
maptable.io.remap_reqs // 更新目标寄存器（ldst → pdst）
maptable.io.ren_br_tags // 分支标签（用于快照）
```

### **RenameFreeList**

管理空闲物理寄存器的分配和释放。

```scala
freelist.io.reqs          // 分配请求（每指令1个）
freelist.io.alloc_pregs   // 分配的物理寄存器（如p60）
freelist.io.dealloc_pregs // 释放的寄存器（提交或回滚时）：
```

优先分配低位寄存器（如 `p1` 比 `p2` 优先）。

避免分配 `p0`（通常固定为0值）。

### **寄存器分配请求**

```scala
ren2_alloc_reqs(w) := ren2_uops(w).ldst_val && ren2_uops(w).dst_rtype === rtype && ren2_fire(w)
```

-  仅当指令需要写入寄存器（`ldst_val`）且类型匹配（整数/浮点）时触发分配。

### **分支恢复**

```scala
remap_reqs(w).ldst := Mux(io.rollback, com.ldst, ren2.ldst)
remap_reqs(w).pdst := Mux(io.rollback, com.stale_pdst, ren2.pdst)
```

-  **正常情况**：更新映射为 `ren2.pdst`（新分配的物理寄存器）。
-  **回滚情况**：恢复为 `com.stale_pdst`（分支前的旧映射）。

### **阻塞信号**

```scala
io.ren_stalls(w) := (ren2_uops(w).dst_rtype === rtype) && !can_allocate
```

### 三张表的具体实现

### RenameMapTable

`RenameMapTable`维护逻辑寄存器到物理寄存器的映射：

1. **核心数据结构**：
   -  `map_table`：当前逻辑到物理寄存器的映射数组
   -  `br_snapshots`：分支预测时保存的快照
2. **主要操作**：
   -  **映射请求**：为每个uop提供逻辑寄存器到物理寄存器的转换
   -  **重映射**：当分配新物理寄存器时更新映射关系
   -  **分支处理**：保存和恢复分支预测时的映射状态
3. **特点**：
   -  支持旁路(bypass)同一周期内前面指令的重命名结果
   -  特殊处理x0寄存器(硬编码为0)

### RenameFreeList

`RenameFreeList`管理可用的物理寄存器：

1. **核心数据结构**：
   -  `free_list`：位向量表示哪些物理寄存器可用
   -  `br_alloc_lists`：为每个分支保存分配的物理寄存器列表
2. **主要操作**：
   -  分配物理寄存器给需要写寄存器的uop
   -  回收已提交指令释放的物理寄存器
   -  处理分支预测错误时的寄存器回收
3. **分配算法**：
   -  使用`SelectFirstN`选择第一个可用的物理寄存器
   -  支持多分配(每个周期最多分配plWidth个寄存器)

### RenameBusyTable

`RenameBusyTable`跟踪物理寄存器的忙状态：

1. **核心数据结构**：
   -  `busy_table`：位向量表示哪些物理寄存器正在被使用
2. **主要功能**：
   -  标记新分配的寄存器为"忙"
   -  当寄存器写回时清除忙状态
   -  提供寄存器忙状态查询
3. **特点**：
   -  支持旁路同一周期内前面指令的分配结果
   -  区分整数和浮点寄存器