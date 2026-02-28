#   Device Tree
## Overview
###  历史背景


**Tips:**



##  设备树

###  背景

早期




设备树（Device Tree）是 Linux 内核中的一种数据结构，用于描述硬件设备信息。设备树通常以树形结构组织，其中每个节点代表一个硬件设备，节点之间通过父子关系连接。设备树可以描述设备的属性、寄存器地址、中断信息、设备树等。

设备树的主要作用是提供硬件设备的信息，以便操作系统和驱动程序能够正确地识别和配置硬件设备。设备树通常以 .dts 文件的形式存在，可以通过编译生成 .dtb 文件，然后在内核启动时加载。

###  设备树的加载方式
（1）由固件负责加载 DTB 并通过标准接口传递（U-Boot 用地址，UEFI 用配置表）；
（2）将 DTB 编译进或追加到内核镜像中。
对于在启动命令行中写 dtb=xxx.dtb 让内核自己去加载文件——不是标准做法，在主线内核中不被支持，
尤其在 RISC-V64 和 LoongArch 上明确不可用，在arm中可用是给内核打了补丁实现的。


####  将设备树嵌入内核


前提：make 编译了某个预定义配置文件后，生成内核顶层的 .config 文件。

1、通过**menuconfig**配置相关开关
```sh
make menuconfig
```
按"/"建，然后键入**DTB**，即可搜索和**DTB**相关的开关即可定位到相关的配置项。形如：
```sh
Symbol: BUILTIN_DTB [=n]
Type  : bool
Defined at arch/riscv/Kconfig:1146
  Prompt: Built-in device tree
  Depends on: OF [=y] && NONPORTABLE [=n]
  Location:
    -> Boot options
(1)    -> Built-in device tree (BUILTIN_DTB [=n])
```
即可知道相关依赖和具体位置，指的类型等信息，再根据实际需要调整即可。

**Tips**: 当填写设备树源码路径时：

* 相对路径基准是 **arch/riscv/boot/dts/**



* 需确保设备树存在于源码目录中（可参考[**添加新的设备树**](#添加新的设备树)），且不需要**dts**后缀。


**eg**:当设备树位于 **arch/riscv/boot/dts/ventana/** 下时,应该填写：

![image](./assets/menuconfig%20DTB%20path.png)

### 添加新的设备树

添加新的设备树到内核中。

1. 在 **arch/riscv/boot/dts/Makefile** 中添加设备树所在目录名（如：ventana）

```sh
subdir-y += ventana
```

**必须做，否则构建系统不会进入 ventana/ 目录。**

2. 创建 ventana/ 子目录及 Makefile

将设备树文件放入 ventana/ 目录，然后创建 Makefile，内容如下：
```sh
dtb-$(CONFIG_ARCH_VENTANA) += thun-v1-2.dtb
```


3. 在 arch/riscv/Kconfig.socs 中添加平台标识

标识需和平台目录中Makefile的宏一致。

```sh
config ARCH_VENTANA
        bool "Ventana SoC family"
        help
          Support for Ventana (T-Head) based SoCs.
```

4. 提供并启用 defconfig

如果平台的defconfig不存在，则需要先创建。
如果平台的defconfig存在，则在其中直接添加一行：
```sh
CONFIG_ARCH_VENTANA=y
```

至此，及完成将平台设备树添加到内核中。










![image](./assets/分区参数.png)












| 阶段              | 是否使用 HOB  | 用途说明                                      |
| --------------- | --------- | ----------------------------------------- |
| **Pre-memory**  | ✅ 是       | 初始构建 HOB，记录早期信息（如临时内存、启动顺序等）              |
| **Post-memory** | ✅ 是       | 添加系统内存信息、模块加载信息、数据迁移后进行更新等                |
| **DXE 阶段开始后**   | ✅ 读取，不再扩展 | DXE Core 解析并利用 HOB 来完成内存、驱动、服务等初始化工作      |
| **DXE 中期以后**    | ❌ 不再使用    | 系统切换为 Boot Services 机制，HOB 完成历史使命不再参与后续流程 |

