---
title: "Linux设备驱动开发详解"
date: 2026-03-15T22:54:00+08:00
draft: false
categories: ["嵌入式开发", "Linux"]
tags: ["Linux", "驱动开发", "内核模块", "字符设备", "设备树"]
math: false
toc: true
readingTime: true
description: "深入讲解Linux设备驱动开发原理、字符设备驱动、平台设备驱动、设备树及完整代码示例"
---

## 概述

Linux设备驱动是连接硬件设备与用户空间程序的桥梁。理解驱动开发是嵌入式Linux工程师的核心技能。

### 驱动分类

| 类型 | 说明 | 示例 |
|------|------|------|
| **字符设备** | 按字节流访问，不可随机读写 | 串口、GPIO、按键 |
| **块设备** | 按块访问，支持随机读写 | 硬盘、SSD、SD卡 |
| **网络设备** | 网络协议栈接口 | 网卡、WiFi |

### 驱动运行层次

```
┌─────────────────────────────────────────┐
│           用户空间应用程序               │
│         (open, read, write, ioctl)       │
├─────────────────────────────────────────┤
│              系统调用接口                │
├─────────────────────────────────────────┤
│           VFS虚拟文件系统               │
├─────────────────────────────────────────┤
│              设备驱动                   │
│    (字符/块/网络设备驱动)               │
├─────────────────────────────────────────┤
│           设备模型与总线                │
│    (platform, i2c, spi, usb等)          │
├─────────────────────────────────────────┤
│              硬件设备                   │
└─────────────────────────────────────────┘
```

---

## 一、内核模块基础

### 1.1 内核模块概述

内核模块（Kernel Module）是可动态加载到内核的代码，无需重新编译整个内核。

**优点：**
- 动态加载/卸载，无需重启
- 减小内核体积
- 方便开发调试

### 1.2 最简内核模块

```c
// hello.c - 最简单的内核模块
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

// 模块加载函数
static int __init hello_init(void)
{
    printk(KERN_INFO "Hello, Kernel!\n");
    return 0;
}

// 模块卸载函数
static void __exit hello_exit(void)
{
    printk(KERN_INFO "Goodbye, Kernel!\n");
}

// 注册模块入口和出口
module_init(hello_init);
module_exit(hello_exit);

// 模块信息
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rovina");
MODULE_DESCRIPTION("A simple kernel module");
MODULE_VERSION("1.0");
```

### 1.3 Makefile编写

```makefile
# Makefile for kernel module

# 模块名称
obj-m += hello.o

# 内核源码路径（根据实际情况修改）
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)

# 默认目标
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

# 清理
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

# 安装模块
install:
	$(MAKE) -C $(KDIR) M=$(PWD) modules_install

.PHONY: all clean install
```

### 1.4 模块操作命令

```bash
# 编译模块
make

# 加载模块
sudo insmod hello.ko

# 查看模块
lsmod | grep hello

# 查看模块信息
modinfo hello.ko

# 查看内核日志
dmesg | tail

# 卸载模块
sudo rmmod hello

# 自动处理依赖加载
sudo modprobe hello
sudo modprobe -r hello
```

### 1.5 模块参数

```c
#include <linux/moduleparam.h>

// 定义参数
static int debug = 0;
static char *name = "default";

// 声明参数
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug, "Debug level (0-3)");

module_param(name, charp, 0644);
MODULE_PARM_DESC(name, "Device name");

// 使用
static int __init my_init(void)
{
    printk(KERN_INFO "debug=%d, name=%s\n", debug, name);
    return 0;
}

// 加载时传递参数
// sudo insmod mymodule.ko debug=1 name="test"
```

---

## 二、字符设备驱动

### 2.1 核心数据结构

```c
// 字符设备结构
struct cdev {
    struct kobject kobj;           // 内嵌的kobject
    struct module *owner;          // 所属模块
    const struct file_operations *ops;  // 文件操作
    struct list_head list;         // 设备链表
    dev_t dev;                     // 设备号
    unsigned int count;            // 设备数量
};

// 设备号
typedef unsigned int dev_t;
#define MAJOR(dev) ((dev) >> 20)   // 主设备号
#define MINOR(dev) ((dev) & 0xfffff) // 次设备号
#define MKDEV(ma, mi) ((ma) << 20 | (mi))  // 构造设备号

// 文件操作结构
struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int (*open)(struct inode *, struct file *);
    int (*release)(struct inode *, struct file *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int (*mmap)(struct file *, struct vm_area_struct *);
    // ... 更多操作
};
```

### 2.2 完整字符设备驱动示例

```c
// mychardev.c - 字符设备驱动完整示例
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/slab.h>

#define DEVICE_NAME "mychardev"
#define CLASS_NAME  "myclass"
#define BUF_SIZE    1024

// 设备私有数据
struct mydev_data {
    char *buffer;          // 数据缓冲区
    size_t size;           // 数据大小
    struct mutex lock;     // 互斥锁
    struct cdev cdev;      // 字符设备结构
};

static dev_t dev_num;               // 设备号
static struct class *dev_class;     // 设备类
static struct device *dev_device;   // 设备
static struct mydev_data *dev_data; // 设备数据

// 打开设备
static int mydev_open(struct inode *inode, struct file *filp)
{
    struct mydev_data *data;
    
    // 从inode获取设备数据
    data = container_of(inode->i_cdev, struct mydev_data, cdev);
    filp->private_data = data;
    
    pr_info("Device opened\n");
    return 0;
}

// 关闭设备
static int mydev_release(struct inode *inode, struct file *filp)
{
    pr_info("Device closed\n");
    return 0;
}

// 读设备
static ssize_t mydev_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *f_pos)
{
    struct mydev_data *data = filp->private_data;
    ssize_t ret = 0;
    
    if (mutex_lock_interruptible(&data->lock))
        return -ERESTARTSYS;
    
    if (*f_pos >= data->size)
        goto out;
    
    if (*f_pos + count > data->size)
        count = data->size - *f_pos;
    
    // 复制数据到用户空间
    if (copy_to_user(buf, data->buffer + *f_pos, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    ret = count;
    
out:
    mutex_unlock(&data->lock);
    return ret;
}

// 写设备
static ssize_t mydev_write(struct file *filp, const char __user *buf,
                           size_t count, loff_t *f_pos)
{
    struct mydev_data *data = filp->private_data;
    ssize_t ret = 0;
    
    if (mutex_lock_interruptible(&data->lock))
        return -ERESTARTSYS;
    
    if (*f_pos >= BUF_SIZE) {
        ret = -ENOSPC;
        goto out;
    }
    
    if (*f_pos + count > BUF_SIZE)
        count = BUF_SIZE - *f_pos;
    
    // 从用户空间复制数据
    if (copy_from_user(data->buffer + *f_pos, buf, count)) {
        ret = -EFAULT;
        goto out;
    }
    
    *f_pos += count;
    if (data->size < *f_pos)
        data->size = *f_pos;
    
    ret = count;
    
out:
    mutex_unlock(&data->lock);
    return ret;
}

// IOCTL操作
#define MYDEV_IOCTL_CLEAR _IO('M', 1)
#define MYDEV_IOCTL_GET_SIZE _IOR('M', 2, int)

static long mydev_ioctl(struct file *filp, unsigned int cmd,
                        unsigned long arg)
{
    struct mydev_data *data = filp->private_data;
    int ret = 0;
    
    switch (cmd) {
    case MYDEV_IOCTL_CLEAR:
        if (mutex_lock_interruptible(&data->lock))
            return -ERESTARTSYS;
        memset(data->buffer, 0, BUF_SIZE);
        data->size = 0;
        mutex_unlock(&data->lock);
        pr_info("Buffer cleared\n");
        break;
        
    case MYDEV_IOCTL_GET_SIZE:
        if (copy_to_user((int __user *)arg, &data->size, sizeof(int)))
            ret = -EFAULT;
        break;
        
    default:
        ret = -ENOTTY;
        break;
    }
    
    return ret;
}

// 文件操作结构
static const struct file_operations mydev_fops = {
    .owner = THIS_MODULE,
    .open = mydev_open,
    .release = mydev_release,
    .read = mydev_read,
    .write = mydev_write,
    .unlocked_ioctl = mydev_ioctl,
};

// 模块初始化
static int __init mydev_init(void)
{
    int ret;
    
    // 1. 分配设备数据
    dev_data = kzalloc(sizeof(*dev_data), GFP_KERNEL);
    if (!dev_data)
        return -ENOMEM;
    
    dev_data->buffer = kzalloc(BUF_SIZE, GFP_KERNEL);
    if (!dev_data->buffer) {
        ret = -ENOMEM;
        goto fail_buffer;
    }
    
    mutex_init(&dev_data->lock);
    
    // 2. 动态分配设备号
    ret = alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    if (ret < 0)
        goto fail_region;
    
    pr_info("Major: %d, Minor: %d\n", MAJOR(dev_num), MINOR(dev_num));
    
    // 3. 初始化字符设备
    cdev_init(&dev_data->cdev, &mydev_fops);
    dev_data->cdev.owner = THIS_MODULE;
    
    ret = cdev_add(&dev_data->cdev, dev_num, 1);
    if (ret < 0)
        goto fail_cdev;
    
    // 4. 创建设备类和设备节点
    dev_class = class_create(CLASS_NAME);
    if (IS_ERR(dev_class)) {
        ret = PTR_ERR(dev_class);
        goto fail_class;
    }
    
    dev_device = device_create(dev_class, NULL, dev_num, NULL, DEVICE_NAME);
    if (IS_ERR(dev_device)) {
        ret = PTR_ERR(dev_device);
        goto fail_device;
    }
    
    pr_info("Device initialized successfully\n");
    return 0;
    
fail_device:
    class_destroy(dev_class);
fail_class:
    cdev_del(&dev_data->cdev);
fail_cdev:
    unregister_chrdev_region(dev_num, 1);
fail_region:
    kfree(dev_data->buffer);
fail_buffer:
    kfree(dev_data);
    return ret;
}

// 模块卸载
static void __exit mydev_exit(void)
{
    device_destroy(dev_class, dev_num);
    class_destroy(dev_class);
    cdev_del(&dev_data->cdev);
    unregister_chrdev_region(dev_num, 1);
    kfree(dev_data->buffer);
    kfree(dev_data);
    
    pr_info("Device removed\n");
}

module_init(mydev_init);
module_exit(mydev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rovina");
MODULE_DESCRIPTION("Character device driver example");
```

### 2.3 用户空间测试程序

```c
// test_dev.c - 用户空间测试程序
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <errno.h>

#define MYDEV_IOCTL_CLEAR _IO('M', 1)
#define MYDEV_IOCTL_GET_SIZE _IOR('M', 2, int)

int main(int argc, char *argv[])
{
    int fd;
    char buf[100];
    ssize_t ret;
    int size;
    
    // 打开设备
    fd = open("/dev/mychardev", O_RDWR);
    if (fd < 0) {
        perror("Failed to open device");
        return 1;
    }
    
    printf("Device opened successfully\n");
    
    // 写入数据
    const char *msg = "Hello from user space!";
    ret = write(fd, msg, strlen(msg));
    if (ret < 0) {
        perror("Write failed");
        close(fd);
        return 1;
    }
    printf("Wrote %zd bytes\n", ret);
    
    // 获取数据大小
    ret = ioctl(fd, MYDEV_IOCTL_GET_SIZE, &size);
    if (ret == 0) {
        printf("Data size: %d bytes\n", size);
    }
    
    // 读取数据
    lseek(fd, 0, SEEK_SET);
    ret = read(fd, buf, sizeof(buf) - 1);
    if (ret < 0) {
        perror("Read failed");
        close(fd);
        return 1;
    }
    buf[ret] = '\0';
    printf("Read: %s\n", buf);
    
    // 清空缓冲区
    ret = ioctl(fd, MYDEV_IOCTL_CLEAR);
    if (ret == 0) {
        printf("Buffer cleared\n");
    }
    
    close(fd);
    return 0;
}
```

---

## 三、并发与竞态处理

### 3.1 自旋锁

```c
#include <linux/spinlock.h>

// 定义自旋锁
spinlock_t my_lock;

// 初始化
spin_lock_init(&my_lock);

// 使用
void spinlock_example(void)
{
    unsigned long flags;
    
    // 普通加锁
    spin_lock(&my_lock);
    // 临界区代码
    spin_unlock(&my_lock);
    
    // 保存中断状态加锁（中断上下文必须）
    spin_lock_irqsave(&my_lock, flags);
    // 临界区代码
    spin_unlock_irqrestore(&my_lock, flags);
}
```

### 3.2 互斥锁

```c
#include <linux/mutex.h>

// 定义互斥锁
struct mutex my_mutex;

// 初始化
mutex_init(&my_mutex);

// 使用
int mutex_example(void)
{
    // 可中断的加锁
    if (mutex_lock_interruptible(&my_mutex))
        return -ERESTARTSYS;
    
    // 临界区代码
    
    mutex_unlock(&my_mutex);
    return 0;
}
```

### 3.3 原子操作

```c
#include <linux/atomic.h>

// 定义原子变量
atomic_t counter = ATOMIC_INIT(0);

// 操作
atomic_inc(&counter);           // 加1
atomic_dec(&counter);           // 减1
atomic_add(5, &counter);        // 加5
atomic_set(&counter, 10);       // 设置值
int val = atomic_read(&counter); // 读取值

// 原子位操作
unsigned long flags;
set_bit(0, &flags);             // 设置位
clear_bit(0, &flags);           // 清除位
change_bit(0, &flags);          // 翻转位
int bit = test_bit(0, &flags);  // 测试位
```

### 3.4 完成量

```c
#include <linux/completion.h>

// 定义
struct completion my_comp;

// 初始化
init_completion(&my_comp);

// 等待完成
wait_for_completion(&my_comp);
// 或超时等待
wait_for_completion_timeout(&my_comp, msecs_to_jiffies(1000));

// 完成通知
complete(&my_comp);           // 唤醒一个等待者
complete_all(&my_comp);       // 唤醒所有等待者
```

---

## 四、中断处理

### 4.1 中断注册

```c
#include <linux/interrupt.h>

// 中断处理函数
static irqreturn_t my_isr(int irq, void *dev_id)
{
    struct mydev_data *data = dev_id;
    
    // 处理中断
    pr_info("Interrupt occurred on IRQ %d\n", irq);
    
    return IRQ_HANDLED;  // 表示中断已处理
}

// 注册中断
int ret = request_irq(irq,                  // 中断号
                      my_isr,               // 处理函数
                      IRQF_TRIGGER_RISING,  // 触发方式
                      "my_device",          // 设备名称
                      dev_data);            // 设备ID

if (ret) {
    pr_err("Failed to request IRQ %d\n", irq);
    return ret;
}

// 释放中断
free_irq(irq, dev_data);
```

### 4.2 中断下半部 - Tasklet

```c
#include <linux/interrupt.h>

// 定义tasklet
void my_tasklet_func(unsigned long data);
DECLARE_TASKLET(my_tasklet, my_tasklet_func, (unsigned long)dev_data);

// tasklet处理函数
void my_tasklet_func(unsigned long data)
{
    struct mydev_data *dev = (struct mydev_data *)data;
    // 延后处理的工作
    pr_info("Tasklet executed\n");
}

// 在中断处理函数中调度
static irqreturn_t my_isr(int irq, void *dev_id)
{
    // 快速处理
    tasklet_schedule(&my_tasklet);
    return IRQ_HANDLED;
}

// 销毁tasklet
tasklet_kill(&my_tasklet);
```

### 4.3 中断下半部 - Workqueue

```c
#include <linux/workqueue.h>

// 定义工作
static struct work_struct my_work;

// 工作处理函数
static void my_work_func(struct work_struct *work)
{
    // 可以睡眠的延后处理
    msleep(100);
    pr_info("Work executed\n");
}

// 初始化
INIT_WORK(&my_work, my_work_func);

// 调度工作
schedule_work(&my_work);

// 延迟调度
schedule_delayed_work(&delayed_work, msecs_to_jiffies(1000));

// 取消工作
cancel_work_sync(&my_work);
```

---

## 五、设备树

### 5.1 设备树基础

设备树（Device Tree）是描述硬件信息的数据结构，用于分离硬件描述和驱动代码。

**设备树文件示例 (mydev.dts)：**

```dts
/dts-v1/;

/ {
    model = "My Development Board";
    compatible = "vendor,myboard";
    
    chosen {
        bootargs = "console=ttyS0,115200";
    };
    
    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x10000000>;  // 256MB
    };
    
    mydevice: mydevice@10000000 {
        compatible = "vendor,mydev";
        reg = <0x10000000 0x1000>;      // 寄存器地址和大小
        interrupts = <0 42 4>;          // 中断号
        clocks = <&clk 0>;
        status = "okay";
        
        my-gpio = <&gpio 10 0>;         // GPIO引用
    };
    
    gpio: gpio@11000000 {
        compatible = "vendor,gpio";
        reg = <0x11000000 0x1000>;
        gpio-controller;
        #gpio-cells = <2>;
    };
};
```

### 5.2 驱动中解析设备树

```c
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_irq.h>
#include <linux/platform_device.h>

static int mydev_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    struct resource res;
    void __iomem *base;
    int irq, ret;
    
    // 获取寄存器资源
    ret = of_address_to_resource(np, 0, &res);
    if (ret) {
        dev_err(&pdev->dev, "Failed to get resource\n");
        return ret;
    }
    
    // 映射寄存器
    base = devm_ioremap_resource(&pdev->dev, &res);
    if (IS_ERR(base))
        return PTR_ERR(base);
    
    // 获取中断号
    irq = irq_of_parse_and_map(np, 0);
    if (irq < 0) {
        dev_err(&pdev->dev, "Failed to get IRQ\n");
        return irq;
    }
    
    // 获取GPIO
    struct gpio_desc *gpiod = devm_gpiod_get(&pdev->dev, "my", GPIOD_OUT_LOW);
    if (IS_ERR(gpiod))
        return PTR_ERR(gpiod);
    
    // 读取属性
    u32 value;
    of_property_read_u32(np, "my-property", &value);
    
    dev_info(&pdev->dev, "Device probed, base=%p, irq=%d\n", base, irq);
    return 0;
}

// 匹配表
static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydev" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

// 平台驱动
static struct platform_driver mydev_driver = {
    .probe = mydev_probe,
    .remove = mydev_remove,
    .driver = {
        .name = "mydev",
        .of_match_table = mydev_of_match,
    },
};

module_platform_driver(mydev_driver);
```

---

## 六、平台设备驱动

### 6.1 平台设备驱动框架

```c
#include <linux/platform_device.h>
#include <linux/mod_devicetable.h>

// 设备私有数据
struct mydev_priv {
    void __iomem *base;
    int irq;
    struct clk *clk;
};

// probe函数 - 设备匹配时调用
static int mydev_probe(struct platform_device *pdev)
{
    struct mydev_priv *priv;
    struct resource *res;
    int ret;
    
    // 分配私有数据
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    
    platform_set_drvdata(pdev, priv);
    
    // 获取内存资源
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    priv->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);
    
    // 获取中断资源
    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0)
        return priv->irq;
    
    // 获取时钟
    priv->clk = devm_clk_get(&pdev->dev, NULL);
    if (IS_ERR(priv->clk))
        return PTR_ERR(priv->clk);
    
    // 使能时钟
    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return ret;
    
    // 注册中断
    ret = devm_request_irq(&pdev->dev, priv->irq, my_isr, 
                           0, dev_name(&pdev->dev), priv);
    if (ret) {
        clk_disable_unprepare(priv->clk);
        return ret;
    }
    
    dev_info(&pdev->dev, "Device probed\n");
    return 0;
}

// remove函数 - 设备移除时调用
static int mydev_remove(struct platform_device *pdev)
{
    struct mydev_priv *priv = platform_get_drvdata(pdev);
    
    clk_disable_unprepare(priv->clk);
    dev_info(&pdev->dev, "Device removed\n");
    return 0;
}

// 设备ID匹配表（传统方式）
static const struct platform_device_id mydev_ids[] = {
    { .name = "mydev", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(platform, mydev_ids);

// 设备树匹配表
static const struct of_device_id mydev_of_match[] = {
    { .compatible = "vendor,mydev", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, mydev_of_match);

// 平台驱动结构
static struct platform_driver mydev_driver = {
    .probe = mydev_probe,
    .remove = mydev_remove,
    .id_table = mydev_ids,
    .driver = {
        .name = "mydev",
        .of_match_table = mydev_of_match,
        .pm = &mydev_pm_ops,  // 电源管理
    },
};

// 模块注册宏
module_platform_driver(mydev_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rovina");
MODULE_DESCRIPTION("Platform device driver example");
```

### 6.2 注册平台设备（板级代码）

```c
// 定义平台资源
static struct resource mydev_resources[] = {
    [0] = {
        .start = 0x10000000,
        .end = 0x10000fff,
        .flags = IORESOURCE_MEM,
    },
    [1] = {
        .start = 42,
        .end = 42,
        .flags = IORESOURCE_IRQ,
    },
};

// 定义平台设备
static struct platform_device mydev_device = {
    .name = "mydev",
    .id = 0,
    .num_resources = ARRAY_SIZE(mydev_resources),
    .resource = mydev_resources,
    .dev = {
        .platform_data = &mydev_pdata,
        .release = mydev_release,
    },
};

// 注册设备
platform_device_register(&mydev_device);

// 卸载时注销
platform_device_unregister(&mydev_device);
```

---

## 七、GPIO驱动

### 7.1 传统GPIO接口

```c
#include <linux/gpio.h>

// 申请GPIO
int ret = gpio_request(10, "my_gpio");
if (ret) {
    pr_err("Failed to request GPIO\n");
    return ret;
}

// 设置方向
gpio_direction_input(10);   // 输入
gpio_direction_output(11, 1); // 输出，初始高电平

// 读写GPIO
int value = gpio_get_value(10);
gpio_set_value(11, 0);

// 映射到IRQ
int irq = gpio_to_irq(10);

// 释放GPIO
gpio_free(10);
```

### 7.2 GPIO描述符接口（推荐）

```c
#include <linux/gpio/consumer.h>

// 获取GPIO描述符
struct gpio_desc *gpiod;

// 从设备树获取
gpiod = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
if (IS_ERR(gpiod))
    return PTR_ERR(gpiod);

// 从索引获取
gpiod = devm_gpiod_get_index(&pdev->dev, NULL, 0, GPIOD_OUT_LOW);

// 设置方向
gpiod_direction_input(gpiod);
gpiod_direction_output(gpiod, 1);

// 读写
int value = gpiod_get_value(gpiod);
gpiod_set_value(gpiod, 1);

// 设置为高阻态
gpiod_set_value_cansleep(gpiod, 0);
gpiod_direction_input(gpiod);

// 自动释放（使用devm_前缀）
// 无需手动调用gpiod_put()
```

### 7.3 GPIO中断

```c
static irqreturn_t gpio_isr(int irq, void *dev_id)
{
    pr_info("GPIO interrupt!\n");
    return IRQ_HANDLED;
}

// 设置GPIO为输入并获取中断
struct gpio_desc *gpiod = devm_gpiod_get(&pdev->dev, "button", GPIOD_IN);
int irq = gpiod_to_irq(gpiod);

// 设置中断触发方式
irq_set_irq_type(irq, IRQ_TYPE_EDGE_RISING);

// 注册中断
devm_request_irq(&pdev->dev, irq, gpio_isr, 
                 IRQF_TRIGGER_RISING, "button", priv);
```

---

## 八、I2C驱动

### 8.1 I2C设备树

```dts
&i2c1 {
    status = "okay";
    clock-frequency = <100000>;
    
    my_i2c_dev: sensor@50 {
        compatible = "vendor,sensor";
        reg = <0x50>;
        interrupt-parent = <&gpio>;
        interrupts = <10 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

### 8.2 I2C驱动框架

```c
#include <linux/i2c.h>

// 设备私有数据
struct sensor_priv {
    struct i2c_client *client;
    struct mutex lock;
};

// 读取寄存器
static int sensor_read_reg(struct i2c_client *client, u8 reg)
{
    return i2c_smbus_read_byte_data(client, reg);
}

// 写入寄存器
static int sensor_write_reg(struct i2c_client *client, u8 reg, u8 val)
{
    return i2c_smbus_write_byte_data(client, reg, val);
}

// 探测函数
static int sensor_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    struct sensor_priv *priv;
    
    priv = devm_kzalloc(&client->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    
    priv->client = client;
    mutex_init(&priv->lock);
    i2c_set_clientdata(client, priv);
    
    // 检测设备ID
    int chip_id = sensor_read_reg(client, 0x00);
    if (chip_id < 0)
        return chip_id;
    
    dev_info(&client->dev, "Sensor detected, ID=0x%02x\n", chip_id);
    return 0;
}

// 移除函数
static void sensor_remove(struct i2c_client *client)
{
    dev_info(&client->dev, "Sensor removed\n");
}

// 设备树匹配
static const struct of_device_id sensor_of_match[] = {
    { .compatible = "vendor,sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, sensor_of_match);

// 传统ID匹配
static const struct i2c_device_id sensor_id[] = {
    { "sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, sensor_id);

// I2C驱动结构
static struct i2c_driver sensor_driver = {
    .probe = sensor_probe,
    .remove = sensor_remove,
    .id_table = sensor_id,
    .driver = {
        .name = "sensor",
        .of_match_table = sensor_of_match,
    },
};

module_i2c_driver(sensor_driver);

MODULE_LICENSE("GPL");
```

---

## 九、调试技术

### 9.1 内核日志

```c
// 日志级别
printk(KERN_EMERG   "Emergency\n");   // 最高优先级
printk(KERN_ALERT   "Alert\n");
printk(KERN_CRIT    "Critical\n");
printk(KERN_ERR     "Error\n");
printk(KERN_WARNING "Warning\n");
printk(KERN_NOTICE  "Notice\n");
printk(KERN_INFO    "Info\n");
printk(KERN_DEBUG   "Debug\n");       // 最低优先级

// 便捷宏
pr_emerg("Emergency\n");
pr_alert("Alert\n");
pr_err("Error: %d\n", ret);
pr_warn("Warning\n");
pr_info("Info: %s\n", name);
pr_debug("Debug: 0x%x\n", value);

// 开发调试
dev_err(&pdev->dev, "Device error\n");
dev_info(&pdev->dev, "Device info\n");
dev_dbg(&pdev->dev, "Device debug\n");
```

### 9.2 动态调试

```bash
# 启用所有调试消息
echo 8 > /proc/sys/kernel/printk

# 动态调试
echo 'module mydev +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file mydev.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'mydev *' > /sys/kernel/debug/dynamic_debug/control

# 查看当前设置
cat /sys/kernel/debug/dynamic_debug/control | grep mydev
```

### 9.3 内核调试工具

```bash
# 查看内核模块信息
cat /proc/modules
cat /sys/module/mydev/parameters/debug

# 查看设备信息
cat /proc/devices
ls -la /sys/class/
ls -la /dev/

# 查看设备树
ls /sys/firmware/devicetree/base/

# 使用debugfs
mount -t debugfs none /sys/kernel/debug
cat /sys/kernel/debug/mydev/status

# 追踪
trace-cmd record -e irq -e sched -p function myapp
trace-cmd report
```

---

## 十、总结

### 10.1 驱动开发流程

```
1. 分析硬件规格 → 确定寄存器、中断、时钟等
2. 编写设备树 → 描述硬件资源
3. 实现驱动框架 → init/exit/file_operations
4. 实现具体功能 → read/write/ioctl等
5. 测试验证 → 用户空间测试程序
6. 优化完善 → 并发、错误处理、电源管理
```

### 10.2 常用API速查

| 功能 | API |
|------|-----|
| 内存分配 | kmalloc, kfree, devm_kzalloc |
| IO映射 | ioremap, iounmap, devm_ioremap |
| 中断 | request_irq, free_irq, devm_request_irq |
| 锁 | spin_lock, mutex_lock, atomic_inc |
| 延时 | udelay, mdelay, msleep |
| DMA | dma_alloc_coherent, dma_map_single |
| GPIO | gpiod_get, gpiod_set_value |
| 时钟 | clk_get, clk_prepare_enable |

### 10.3 学习资源

- Linux内核源码 Documentation/
- Linux Device Drivers 3 (LDD3)
- Essential Linux Device Drivers
- 内核源码 drivers/ 目录下的示例驱动

---

## 参考资料

- Linux Kernel Documentation: https://www.kernel.org/doc/
- Linux Device Drivers 3: https://lwn.net/Kernel/LDD3/
- Device Tree Specification: https://www.devicetree.org/
- Kernel Newbies: https://kernelnewbies.org/