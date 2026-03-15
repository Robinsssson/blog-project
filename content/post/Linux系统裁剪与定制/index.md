---
title: "Linux系统裁剪与定制详解"
date: 2026-03-15T22:43:00+08:00
draft: false
categories: ["嵌入式开发", "Linux"]
tags: ["Linux", "内核裁剪", "BusyBox", "嵌入式", "系统定制"]
math: false
toc: true
readingTime: true
description: "深入讲解Linux系统裁剪原理、内核配置、根文件系统精简及完整实践流程"
---

## 概述

Linux系统裁剪是指根据实际需求，移除不必要的组件，精简系统体积，优化启动速度和运行性能。这在嵌入式设备、工业控制、物联网等领域有广泛应用。

### 为什么需要裁剪？

| 场景 | 原因 |
|------|------|
| **嵌入式设备** | 存储空间有限（几MB到几百MB） |
| **工业控制** | 追求快速启动和实时响应 |
| **安全加固** | 减少攻击面，移除不必要的服务 |
| **云原生** | 容器镜像最小化，加快部署速度 |

### 裁剪目标

- **体积精简**：从几GB减少到几十MB甚至几MB
- **启动加速**：从分钟级减少到秒级
- **资源优化**：减少CPU、内存占用
- **安全加固**：最小化攻击面

---

## 一、Linux系统组成

### 1.1 系统层次结构

```
┌─────────────────────────────────────────────┐
│              用户空间应用                     │
│  (Shell, 工具, 服务, GUI等)                  │
├─────────────────────────────────────────────┤
│              系统库 (Libraries)              │
│  (glibc, libpthread, libm等)                │
├─────────────────────────────────────────────┤
│              系统调用接口                    │
│              (System Calls)                 │
├─────────────────────────────────────────────┤
│              Linux内核                       │
│  (进程管理, 内存管理, 驱动, 网络协议栈等)     │
├─────────────────────────────────────────────┤
│              硬件平台                        │
│  (CPU, 内存, 存储, 外设)                     │
└─────────────────────────────────────────────┘
```

### 1.2 可裁剪的组件

| 层次 | 组件 | 裁剪方法 |
|------|------|----------|
| **应用层** | 不必要的工具、服务 | 移除软件包 |
| **库层** | 未使用的库函数 | 使用精简版库（musl、uClibc） |
| **内核层** | 未使用的驱动、功能 | 内核配置裁剪 |
| **启动层** | 引导加载器 | 精简U-Boot/GRUB |

---

## 二、内核裁剪

### 2.1 内核配置系统

Linux内核使用Kconfig系统进行配置：

```bash
# 进入内核源码目录
cd linux-x.y.z

# 配置方式
make menuconfig    # 图形化菜单配置（推荐）
make config        # 逐项问答式配置
make xconfig       # Qt图形界面
make defconfig     # 使用默认配置
make allnoconfig   # 全部禁用，最小化配置
```

### 2.2 内核配置选项

**关键配置目录：**

```
kernel/
├── General setup          # 通用设置
├── Processor type         # 处理器类型
├── Power management       # 电源管理
├── Bus options            # 总线选项
├── Device Drivers         # 设备驱动（重点裁剪区域）
├── File systems           # 文件系统
├── Networking             # 网络协议栈
├── Security options       # 安全选项
└── Kernel hacking         # 调试选项
```

### 2.3 内核裁剪实践

```bash
#!/bin/bash
# 内核裁剪脚本示例

# 1. 获取硬件信息
lspci                    # PCI设备
lsusb                    # USB设备
cat /proc/cpuinfo        # CPU信息
cat /proc/meminfo        # 内存信息

# 2. 查看当前内核配置
zcat /proc/config.gz > .config  # 从当前运行的内核导出配置

# 3. 使用localmodconfig自动裁剪
# 只保留当前加载的模块
make localmodconfig

# 4. 手动精简配置
make menuconfig
```

### 2.4 关键裁剪选项

**最小化内核配置建议：**

```kconfig
# 通用设置
CONFIG_LOCALVERSION="-custom"
CONFIG_LOCALVERSION_AUTO=n

# 移除不需要的功能
CONFIG_MODULES=n              # 禁用模块（嵌入式推荐）
CONFIG_BUG=n                  # 移除BUG检查
CONFIG_FUTEX=n                # 禁用futex（无多线程需求时）
CONFIG_EPOLL=n                # 禁用epoll（无事件驱动需求时）

# 处理器优化
CONFIG_SMP=n                  # 单核CPU禁用SMP
CONFIG_PREEMPT_NONE=y         # 无抢占（服务器）
CONFIG_PREEMPT=y              # 抢占式（嵌入式/实时）

# 电源管理（嵌入式通常禁用）
CONFIG_PM=n
CONFIG_ACPI=n
CONFIG_SUSPEND=n

# 内存管理
CONFIG_SWAP=n                 # 无交换分区时禁用
CONFIG_MEMORY_HOTPLUG=n       # 禁用内存热插拔

# 文件系统（只保留需要的）
CONFIG_EXT4_FS=y              # 根文件系统
CONFIG_FAT_FS=n               # 无FAT需求
CONFIG_NTFS_FS=n              # 无NTFS需求
CONFIG_PROC_FS=y              # 必须保留
CONFIG_SYSFS=y                # 必须保留
CONFIG_DEVPTS_FS=y            # 必须保留（PTY）

# 网络协议栈（大幅裁剪）
CONFIG_NET=n                  # 无网络需求
# 或只保留必要的
CONFIG_INET=y
CONFIG_TCP_CONG_CUBIC=y
CONFIG_IPV6=n                 # 禁用IPv6
CONFIG_WIRELESS=n             # 禁用无线
CONFIG_BT=n                   # 禁用蓝牙

# 设备驱动（重点裁剪）
# 只保留实际使用的硬件驱动
CONFIG_BLK_DEV_SD=y           # SCSI/SATA磁盘
CONFIG_ATA=y                  # ATA驱动
CONFIG_SERIAL_8250=y          # 串口
CONFIG_GPIO=y                 # GPIO

# 移除不需要的驱动
CONFIG_SOUND=n                # 禁用声卡
CONFIG_MEDIA_SUPPORT=n        # 禁用多媒体
CONFIG_DRM=n                  # 禁用DRM图形
CONFIG_USB=n                  # 无USB设备时禁用
CONFIG_I2C=n                  # 无I2C设备时禁用

# 调试选项（生产环境禁用）
CONFIG_DEBUG_KERNEL=n
CONFIG_DEBUG_INFO=n
CONFIG_KALLSYMS=n
CONFIG_PRINTK=n               # 可选禁用打印
```

### 2.5 内核编译

```bash
# 编译内核
make -j$(nproc)

# 编译模块（如果启用）
make modules_install

# 安装内核
make install

# 单独编译设备树（ARM嵌入式）
make dtbs

# 输出文件
# vmlinux       - 未压缩的内核镜像（ELF格式）
# arch/x86/boot/bzImage - 压缩的可启动镜像（x86）
# arch/arm/boot/zImage - ARM压缩镜像
# arch/arm/boot/dts/*.dtb - 设备树二进制
```

### 2.6 内核体积分析

```bash
# 查看内核各部分大小
size vmlinux

# 查看符号表大小
nm --size-sort vmlinux | tail -20

# 分析内核配置
scripts/config --set-val CONFIG_DEBUG_INFO n

# 估算裁剪后大小
du -h vmlinux
```

---

## 三、根文件系统精简

### 3.1 BusyBox：嵌入式瑞士军刀

BusyBox将多个常用Unix工具合并到一个可执行文件中：

```bash
# BusyBox包含的工具
/bin/busybox --list | head -20
# 输出示例：
# [, [[, addgroup, adduser, ar, arch, ash, awk, base64, basename
# bbconfig, beep, blkdiscard, blkid, blockdev, bootchartd, brctl
# bunzip2, bzcat, bzip2, cal, cat, chat, chattr, chgrp, chmod

# 体积对比
ls -lh /bin/busybox        # 约 1-2 MB
ls -lh /bin/* | wc -l      # 标准系统可能有上百个独立工具
```

**BusyBox配置：**

```bash
# 获取BusyBox
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar xjf busybox-1.36.0.tar.bz2
cd busybox-1.36.0

# 配置
make defconfig             # 默认配置
make menuconfig            # 自定义配置

# 关键配置选项
# Settings ->
#   Build Options ->
#     Build static binary (no shared libs)  # 静态编译
#   Installation Options ->
#     Don't use /usr                         # 安装路径

# 编译
make -j$(nproc)

# 安装
make CONFIG_PREFIX=/path/to/rootfs install
```

### 3.2 最小根文件系统结构

```bash
#!/bin/bash
# 创建最小根文件系统

ROOTFS=/opt/rootfs

# 创建目录结构
mkdir -p $ROOTFS/{bin,sbin,etc,proc,sys,dev,tmp,var,root,usr/bin,usr/sbin}

# 安装BusyBox
make CONFIG_PREFIX=$ROOTFS install

# 创建设备节点
cd $ROOTFS/dev
sudo mknod console c 5 1
sudo mknod null c 1 3
sudo mknod zero c 1 5
sudo mknod tty c 5 0
sudo mknod tty0 c 4 0
sudo mknod tty1 c 4 1

# 创建基本配置文件
cd $ROOTFS/etc

# inittab - init进程配置
cat > inittab << 'EOF'
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
EOF

# fstab - 文件系统挂载表
cat > fstab << 'EOF'
proc    /proc   proc    defaults    0 0
sysfs   /sys    sysfs   defaults    0 0
devpts  /dev/pts devpts defaults    0 0
tmpfs   /tmp    tmpfs   defaults    0 0
EOF

# init.d/rcS - 启动脚本
mkdir -p init.d
cat > init.d/rcS << 'EOF'
#!/bin/sh
mount -a
echo "Welcome to Mini Linux"
hostname mini-linux
EOF
chmod +x init.d/rcS

# passwd - 用户配置
cat > passwd << 'EOF'
root:x:0:0:root:/root:/bin/sh
EOF

# group - 组配置
cat > group << 'EOF'
root:x:0:
EOF

# 设置权限
chmod +x $ROOTFS/init.d/rcS
```

### 3.3 C库选择与裁剪

| C库 | 大小 | 特点 | 适用场景 |
|------|------|------|----------|
| **glibc** | ~2MB | 功能完整，兼容性好 | 桌面/服务器 |
| **musl** | ~400KB | 轻量，静态链接友好 | 容器/嵌入式 |
| **uClibc-ng** | ~200KB | 可配置，极小 | 深度嵌入式 |
| **newlib** | ~100KB | 面向裸机 | 无OS环境 |

**musl库编译示例：**

```bash
# 下载musl
wget https://musl.libc.org/releases/musl-1.2.4.tar.gz
tar xzf musl-1.2.4.tar.gz
cd musl-1.2.4

# 配置
./configure --prefix=/opt/musl --disable-shared

# 编译安装
make -j$(nproc)
make install

# 使用musl编译程序
/opt/musl/bin/musl-gcc -static -o myapp myapp.c
```

### 3.4 使用Buildroot构建

Buildroot是自动化构建嵌入式Linux系统的工具：

```bash
# 获取Buildroot
wget https://buildroot.org/downloads/buildroot-2024.02.tar.gz
tar xzf buildroot-2024.02.tar.gz
cd buildroot-2024.02

# 配置
make menuconfig

# 关键配置：
# Target options -> Target Architecture (ARM, x86等)
# Toolchain -> C library (musl/uClibc)
# System configuration -> Root filesystem overlay
# Filesystem images -> ext4/cpio/tar

# 编译（会自动下载编译所有组件）
make -j$(nproc)

# 输出文件
# output/images/rootfs.tar
# output/images/rootfs.ext4
# output/images/zImage
# output/images/*.dtb
```

**Buildroot最小配置示例：**

```
# .config 片段
BR2_arm=y                          # ARM架构
BR2_cortex_a7=y                    # Cortex-A7
BR2_TOOLCHAIN_BUILDROOT_MUSL=y    # 使用musl C库
BR2_TARGET_GENERIC_ROOT_PASSWD=""  # 无密码
BR2_TARGET_GENERIC_GETTY_PORT="ttyS0"
BR2_ROOTFS_OVERLAY="overlay/"      # 自定义文件覆盖
BR2_PACKAGE_BUSYBOX=y              # BusyBox
BR2_PACKAGE_BUSYBOX_CONFIG="busybox.config"
BR2_TARGET_ROOTFS_CPIO=y           # cpio格式
BR2_TARGET_ROOTFS_EXT2=y           # ext2格式
```

---

## 四、系统启动优化

### 4.1 启动流程分析

```
┌─────────────────────────────────────────────────────────┐
│                    Linux启动流程                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐                                       │
│  │ ROM/BIOS     │  硬件初始化，加载引导程序              │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Bootloader   │  U-Boot/GRUB，加载内核                │
│  │ (U-Boot)     │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ Linux Kernel │  内核初始化，挂载根文件系统            │
│  │              │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ /sbin/init   │  用户空间初始化                       │
│  │ (PID 1)      │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│         ▼                                               │
│  ┌──────────────┐                                       │
│  │ 用户应用     │  运行目标应用程序                      │
│  └──────────────┘                                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.2 内核启动参数优化

```bash
# U-Boot启动参数示例
setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 \
                 rootwait rw init=/sbin/init \
                 quiet loglevel=0'

# 关键参数说明：
# console=ttyS0,115200  - 控制台设备
# root=/dev/mmcblk0p2   - 根文件系统设备
# rootfstype=ext4       - 根文件系统类型
# rootwait              - 等待根设备就绪
# rw                    - 读写挂载
# init=/sbin/init       - init程序路径
# quiet                 - 静默模式（减少启动输出）
# loglevel=0            - 日志级别
# initcall_debug        - 调试init调用（调试用）
```

### 4.3 Init系统选择

| Init系统 | 特点 | 启动速度 | 适用场景 |
|----------|------|----------|----------|
| **systemd** | 功能强大，并行启动 | 快 | 桌面/服务器 |
| **SysV init** | 传统，串行启动 | 慢 | 传统系统 |
| **BusyBox init** | 轻量，简单 | 最快 | 嵌入式 |
| **OpenRC** | 模块化，脚本化 | 中等 | Gentoo等 |

**BusyBox init最小配置：**

```bash
# /etc/inittab
::sysinit:/etc/init.d/rcS        # 系统初始化
ttyS0::askfirst:-/bin/sh         # 串口登录
::ctrlaltdel:/sbin/reboot        # Ctrl+Alt+Del重启
::shutdown:/sbin/swapoff -a      # 关闭交换分区
::shutdown:/bin/umount -a -r     # 卸载文件系统

# /etc/init.d/rcS
#!/bin/sh
mount -a                         # 挂载所有文件系统
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
echo "System started"
```

### 4.4 启动时间分析

```bash
# systemd-analyze分析启动时间
systemd-analyze time            # 总启动时间
systemd-analyze blame           # 各服务启动时间
systemd-analyze critical-chain  # 关键启动链

# 内核启动时间分析
# 添加内核参数：initcall_debug
dmesg | grep "initcall" | head -20

# 使用ftrace分析
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... 系统启动 ...
cat /sys/kernel/debug/tracing/trace
```

---

## 五、实用裁剪案例

### 5.1 案例一：树莓派最小系统

**目标：** 16MB SD卡可运行的Linux系统

```bash
# 1. 使用Buildroot构建
make raspberrypi4_defconfig
make menuconfig

# 配置选项：
# - 禁用所有图形相关
# - 禁用音频
# - 禁用USB（如果不需要）
# - 使用musl C库
# - 精简BusyBox配置

# 2. 内核裁剪
# 移除：
# - GPU驱动（使用headless模式）
# - 音频驱动
# - 摄像头驱动
# - WiFi/蓝牙驱动

# 3. 根文件系统
# 大小估算：
# - 内核: ~5MB
# - BusyBox: ~1MB
# - musl: ~0.5MB
# - 基本配置: ~0.1MB
# - 总计: ~7MB
```

### 5.2 案例二：嵌入式网关系统

**需求：** 工业网关，需要网络、串口、Modbus

```bash
# 1. 内核配置
# 保留：
CONFIG_NET=y
CONFIG_INET=y
CONFIG_SERIAL_8250=y
CONFIG_SPI=y           # Modbus可能用SPI
CONFIG_I2C=y           # 可能用I2C传感器

# 移除：
CONFIG_SOUND=n
CONFIG_MEDIA_SUPPORT=n
CONFIG_DRM=n
CONFIG_USB=n           # 无USB需求

# 2. 用户空间
# 必要程序：
# - BusyBox（基础工具）
# - dropbear（轻量SSH）
# - mosquitto（MQTT代理）
# - socat（串口转发）

# 3. 估算大小
# - 内核: ~4MB
# - 根文件系统: ~10MB
# - 应用程序: ~5MB
# - 总计: ~20MB
```

### 5.3 案例三：容器基础镜像

**目标：** 最小Docker基础镜像

```dockerfile
# Dockerfile
FROM scratch

# 使用静态编译的BusyBox
ADD busybox /

# 创建目录结构
RUN ["/busybox", "mkdir", "-p", "/bin", "/sbin", "/etc", "/proc", "/sys"]
RUN ["/busybox", "ln", "-s", "/busybox", "/bin/sh"]

# 设置环境
ENV PATH=/bin:/sbin

CMD ["/bin/sh"]
```

```bash
# 编译静态BusyBox
make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc)

# 镜像大小
# scratch + busybox = ~1MB
```

---

## 六、裁剪工具与技巧

### 6.1 常用工具

```bash
# 分析可执行文件依赖
ldd /bin/ls
readelf -d /bin/ls | grep NEEDED

# 查看动态库导出符号
nm -D /lib/x86_64-linux-gnu/libc.so.6

# 剥离调试符号
strip --strip-all /usr/bin/myapp

# 压缩可执行文件
upx --best /usr/bin/myapp

# 分析文件大小贡献
size /usr/bin/*
```

### 6.2 依赖分析脚本

```bash
#!/bin/bash
# 分析程序及其依赖的总大小

analyze_deps() {
    local prog=$1
    local total=0
    local deps=$(ldd "$prog" 2>/dev/null | grep "=> /" | awk '{print $3}')
    
    echo "=== $prog ==="
    
    # 程序本身大小
    local prog_size=$(stat -c%s "$prog")
    total=$((total + prog_size))
    echo "Binary: $prog_size bytes"
    
    # 依赖库大小
    for dep in $deps; do
        local lib_size=$(stat -c%s "$dep" 2>/dev/null)
        if [ -n "$lib_size" ]; then
            total=$((total + lib_size))
            echo "  $(basename $dep): $lib_size bytes"
        fi
    done
    
    echo "Total: $total bytes ($(numfmt --to=iec $total))"
}

analyze_deps /bin/ls
```

### 6.3 内核模块精简

```bash
# 查看当前加载的模块
lsmod

# 查看模块依赖
modinfo -F depends module_name

# 永久移除模块
echo "blacklist module_name" >> /etc/modprobe.d/blacklist.conf

# 更新initramfs（如果使用）
update-initramfs -u
```

---

## 七、完整实践：构建最小Linux系统

### 7.1 目标规格

- 平台：x86_64虚拟机
- 存储限制：10MB
- 功能：串口控制台、基本shell命令
- 启动时间：< 5秒

### 7.2 实施步骤

```bash
#!/bin/bash
# 最小Linux系统构建脚本

set -e

# ===== 1. 准备工作 =====
WORKDIR=/opt/minilinux
ROOTFS=$WORKDIR/rootfs
mkdir -p $WORKDIR $ROOTFS

# ===== 2. 编译内核 =====
cd $WORKDIR
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.tar.xz
tar xf linux-6.6.tar.xz
cd linux-6.6

# 使用最小配置
make allnoconfig

# 添加必要选项
./scripts/config --enable CONFIG_64BIT
./scripts/config --enable CONFIG_PRINTK
./scripts/config --enable CONFIG_TTY
./scripts/config --enable CONFIG_SERIAL_8250
./scripts/config --enable CONFIG_SERIAL_8250_CONSOLE
./scripts/config --enable CONFIG_BINFMT_ELF
./scripts/config --enable CONFIG_BINFMT_SCRIPT
./scripts/config --enable CONFIG_PROC_FS
./scripts/config --enable CONFIG_SYSFS
./scripts/config --enable CONFIG_DEVTMPFS
./scripts/config --enable CONFIG_DEVTMPFS_MOUNT
./scripts/config --enable CONFIG_EXT4_FS
./scripts/config --enable CONFIG_VT
./scripts/config --enable CONFIG_VT_CONSOLE

make olddefconfig
make -j$(nproc)

# 内核大小
ls -lh arch/x86/boot/bzImage
# 约 1-2 MB

# ===== 3. 编译BusyBox =====
cd $WORKDIR
wget https://busybox.net/downloads/busybox-1.36.0.tar.bz2
tar xf busybox-1.36.0.tar.bz2
cd busybox-1.36.0

make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
make -j$(nproc)
make CONFIG_PREFIX=$ROOTFS install

# BusyBox大小
ls -lh $ROOTFS/bin/busybox
# 约 1-2 MB

# ===== 4. 创建根文件系统 =====
cd $ROOTFS

# 目录结构
mkdir -p {dev,proc,sys,etc,tmp,root}

# 设备节点
sudo mknod -m 622 dev/console c 5 1
sudo mknod -m 666 dev/null c 1 3
sudo mknod -m 666 dev/zero c 1 5
sudo mknod -m 666 dev/tty c 5 0

# init脚本
cat > init << 'EOF'
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
echo "Welcome to Mini Linux!"
exec /bin/sh
EOF
chmod +x init

# ===== 5. 创建initramfs =====
cd $ROOTFS
find . | cpio -o -H newc | gzip > $WORKDIR/initramfs.cpio.gz

# initramfs大小
ls -lh $WORKDIR/initramfs.cpio.gz
# 约 1-2 MB

# ===== 6. 使用QEMU测试 =====
qemu-system-x86_64 \
    -kernel $WORKDIR/linux-6.6/arch/x86/boot/bzImage \
    -initrd $WORKDIR/initramfs.cpio.gz \
    -append "console=ttyS0" \
    -nographic \
    -m 64M

# ===== 7. 总大小统计 =====
echo "=== 系统大小 ==="
echo "Kernel: $(du -h $WORKDIR/linux-6.6/arch/x86/boot/bzImage | cut -f1)"
echo "RootFS: $(du -h $WORKDIR/initramfs.cpio.gz | cut -f1)"
echo "Total:  $(du -sh $WORKDIR/linux-6.6/arch/x86/boot/bzImage $WORKDIR/initramfs.cpio.gz | tail -1)"
```

### 7.3 预期结果

```
=== 系统大小 ===
Kernel: 1.2M
RootFS: 1.5M
Total:  2.7M

# 可以轻松放入10MB存储空间
```

---

## 八、总结

### 8.1 裁剪原则

1. **明确需求**：确定系统需要哪些功能
2. **逐步精简**：从大而全到小而精
3. **测试验证**：每步裁剪后测试功能
4. **文档记录**：记录裁剪内容，便于维护

### 8.2 裁剪效果对比

| 项目 | 裁剪前 | 裁剪后 | 减少 |
|------|--------|--------|------|
| 内核大小 | ~10MB | ~2MB | 80% |
| 根文件系统 | ~500MB | ~3MB | 99% |
| 启动时间 | ~30s | ~3s | 90% |
| 内存占用 | ~200MB | ~20MB | 90% |

### 8.3 工具链选择

| 需求 | 推荐工具 |
|------|----------|
| 快速原型 | Buildroot |
| 生产系统 | Yocto |
| 容器镜像 | Alpine/musl |
| 学习研究 | 手动构建 |

---

## 参考资料

- Linux Kernel Documentation: https://www.kernel.org/doc/
- Buildroot Manual: https://buildroot.org/downloads/manual/manual.html
- BusyBox Documentation: https://busybox.net/FAQ.html
- Yocto Project: https://docs.yoctoproject.org/
- Embedded Linux Wiki: https://elinux.org/Main_Page