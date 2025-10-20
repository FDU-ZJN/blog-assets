---
title: 适配AXI的地址对齐
categories: AXI
tags:
  - IC
cover: /img/cover_8.jpg
highlight_shrink: true
abbrlink: 3521918645
date: 2025-04-11 11:21:41
---

## AXI4引入的arsize/awsize

完整的AXI总线协议通过`arsize`/`awsize`信号来指示实际访问的数据位宽, 同时引入"窄传输"的概念, 用于指示"实际数据位宽小于总线数据位宽"的情况. 这两个"数据位宽"的概念并非完全一致, 具体地, 总线数据位宽是在硬件设计时静态决定的, 它表示一次总线传输的最大数据位宽, 也用于计算总线的理论带宽; 而实际数据位宽(即`arsize`/`awsize`信号的值)则由软件访存指令中的位宽信息动态决定, 它表示一次总线传输的实际数据位宽, 例如, `lb`指令只访问1字节, 而`lw`指令则访问4字节。

也就是AXI总线会根据我们设置的`arsize`/`awsize`来映射对齐地址，设置为0时，会对齐`1b`,设置为1时，会对齐`2b`,设置为4时，会对齐`4b`。

## strb选通信号与data

### S指令有效数据与地址的对齐

`wstrb`与地址相适配，也要将有效数据移至有效字节，以`sb`指令为例

```scala
      is(EXE_SB_OP) {
        mem_addr := io.ex2mem.mem_addr
        mem_ce := true.B
        switch(io.ex2mem.mem_addr(1,0)) {
          is(0.U) { mem_sel := "b0001".U }
          is(1.U) { mem_sel := "b0010".U } 
          is(2.U) { mem_sel := "b0100".U }
          is(3.U) { mem_sel := "b1000".U } 
        }
        io.mem2ram.mem_awsize := 0.U
        mem_data := io.ex2mem.reg2 << (Cat(io.ex2mem.mem_addr(1,0), 0.U(3.W))) 
      }
```

### L指令截取有效数据

r通道不存在选通信号，需根据地址截取，以`lw`指令为例

```scala
      is(EXE_LBU_OP) {
        mem_addr := io.ex2mem.mem_addr
        mem_ce := true.B
        val byte_data = MuxLookup(io.ex2mem.mem_addr(1, 0), 0.U(8.W), Seq(
        0.U -> io.mem2ram.mem_data_i(7, 0),    
        1.U -> io.mem2ram.mem_data_i(15, 8),   
        2.U -> io.mem2ram.mem_data_i(23, 16),  
        3.U -> io.mem2ram.mem_data_i(31, 24)  
      ))
        io.mem2ram.mem_arsize := 0.U
        mem_wdata := Cat(Fill(24, 0.U(1.W)), byte_data(7, 0))
      }
```

## 适配AXI4的设备

一个设备如果要适配AXI4总线，最重要的就是地址对齐

对于32位存储器，输入其地址必须对齐4b，如rom的软件实现

```c
extern "C" void mrom_read(int32_t addr, int32_t *data) { *data = host_read(pmem+(addr&~0x3)-0x20000000L,4); }
```

这样他输出的32位数据就自动是对齐地址，可供lb,lh指令截取

对于8位1字节设备，对齐地址需要将有效数据移至地址对应字节，如`uart`设备，其状态寄存器位于相对地址`5`，就需要将1字节的数据放在32位数据的[15:8]，当然，最简单的方法就是将这个字节复制4次塞在32位数据中。

唉，debug了半天，其实整理下来思路还是挺简单的，就是无论如何地址要对齐，但总是思路就卡在哪里了。

