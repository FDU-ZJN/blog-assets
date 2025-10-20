---
title: ld链接与简易bootloader（添加全局变量支持）
categories: 编译
tags:
  - 计算机
cover: /img/cover_7.jpg
highlight_shrink: true
abbrlink: 132441525
date: 2025-04-08 17:47:55
---

#### 堆区

在启动文件里分配，作为用户主动申请时的空间，如调用malloc()

#### 栈区

在启动文件里分配，作为局部变量自动申请和释放空间的变量（也有说是编译器分配的空间）

#### bss

存放未初始化的全局变量和静态变量；

#### data

存放初始化后的全局变量和静态变量；

![ab4c927f0fd66f2b12e8b8638e5f054a](/images/ld链接与简易bootloader/ab4c927f0fd66f2b12e8b8638e5f054a.png)

已初始化的全局变量（data）嵌入程序，不可能直接存在RAM中，必须先载入ROM，再通过memcpy载入RAM中（这也是数电H中我没能想到的，不过那时也没实现标准库也不太可能搞这种骚操作），bss由于是未初始化的，直接链接到RAM就行。

```c
  _sidata = LOADADDR(.data);
  .data : {
    _sdata = .;
    *(.data*)
    *(.sdata*)
    _edata = .;
  } > sram AT >mrom :data

 .bss : 
  {
      _bstart = .;
    *(.bss*)
    *(.sbss*)
    *(.scommon)
    _bend = .;
  } > sram
```

`>`后表示VMA在的存储器区间, `AT>`后表示LMA所在的存储器区间,这样表示开始存在ROM中，但链接到了RAM中，要调用会自动跑到RAM中调用。

那么下一步就是想办法整到ram中，原来叫bootloader，我说那时单片机有个这东西也不知道什么东西。

地址是向下增长的，我们需要`.data` 段的**初始值**在 **ROM** 中的**起始地址** (LMA)，

通过

```
_sidata = LOADADDR(.data)
```

而

-  `_sdata`: `.data` 段在 **RAM** 中的**起始地址** (VMA)。
-  `_edata`: `.data` 段在 **RAM** 中的**结束地址** (VMA)。

那么万事俱备，前后加载地址我们都得到了，开始bootloader吧

```c
  extern uint8_t _sdata[], _edata[], _sidata[];
  extern uint8_t _bstart[], _bend[];

  uint32_t data_size = _edata - _sdata;
  memcpy(_sdata, _sidata, data_size);
```

_sidata是LMA起始地址，我们只需向后截取ROM中data段长度的数据，转至RAM，memcpy是最舒服的

bss段就不需要移了，本来就在RAM直接分配了，但必须还需要初始化

```c
  uint32_t bss_size = _bend - _bstart;

  memset(_bstart, 0, bss_size);
```

高贵的标准库函数很便利地实现。

好的，这样简单的操作就能让c代码支持全局变量了，这个移取的思路简单却很难想到，不然数电荣誉课那时应该能玩的更花了。