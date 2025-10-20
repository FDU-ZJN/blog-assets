---
title: Arch的ssh连接，远程桌面，vivado安装
categories: Linux
tags:
  - 计算机
cover: /img/cover_10.jpg
highlight_shrink: true
abbrlink: 3740229499
date: 2025-06-18 10:38:40
---

内容均参考官方wiki

## ssh连接

### **(1) 安装 `openssh`**

```
sudo pacman -S openssh
```

### **(2) 启动 SSH 服务**

```
sudo systemctl enable --now sshd
```

现在我使用winscp，复制文件很方便

## 远程桌面

### 局域网内连接

xrdp试了很多次，一直黑屏

目前更换成了tigervnc，配置参考wiki，但还是会黑屏，因此使用共享物理桌面

```
x0vncserver -display :0 -passwordfile ~/.vnc/passwd
```

用tigervnc应用连接 ip::5900(:0是偏移，即端口为5900+0)

### 跨局域网连接

首先获得路由器网络公网ip

```
curl ifconfig.me
```

注意公网ip会2-3天一换

在浏览器输入路由器管理网址，配置端口转发

如我的端口是5900，我设置外端口为25900，内端口为5900，ip就是局域网ip

通过tigervnc应用连接公网ip::25900

连接的屏幕只有大概原本屏幕的2/3，需要拖拽

目前能用就行，后续会继续探索更高效的方法

## Vivado 安装

官方文档很落后，只到2020.3，我尝试安装的是2023.2

安装兼容库

```
paru -S ncurses5-compat-libs
```

下载 **AMD Unified Installer for FPGAs & Adaptive SoCs 2023.2: Linux **

选择安装路径为 **`/home/<我的用户名>/vivado`**（也可考虑使用隐藏路径或 `/opt`）

运行安装程序

```
vivado
```

实际上安装完有几个系统上的小问题，如我配置时没有生成**locale**,缺少兼容库等，问一下AI能很快解决

运行速度明显比在windows上快很多