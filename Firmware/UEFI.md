#   UEFI
## Overview
###  历史背景
“Legacy BIOS（传统的 BIOS）”缺点，不足：
*   安全隐患：无 Secure Boot 等机制，易受引导区病毒攻击。
*   未标准化：不同厂商的 BIOS 实现差异大，兼容性差。
*   拓展性差：Legacy BIOS 功能固化，缺乏动态扩展能力，新增硬件支持通常需主板厂商发布新版固件。

**Tips:**

“新硬件支持”特指：在 Legacy BIOS 启动流程中（POST 到 bootloader 加载前），需要被初始化、识别或用于引导操作系统的新型硬件组件。
主要包括：
新型存储设备（NVMe、USB 3.x 启动）
新 CPU/芯片组（需微码和初始化序列）
新输入/显示接口（需 CSM 兼容支持）。

###  UEFI 定义
UEFI（Unified Extensible Firmware Interface，统一可扩展固件接口）定义了**操作系统和硬件体系之间的接口标准**。是一种约定规范，不包含具体实现；目前主流的开源实现是**TianoCore的EDK II**，由Intel提供的。

### UEFI标准概述
启动管理器（Boot Manager）：OS或UEFI shell等UEFI 应用程序，可以在任何符合规范（指硬件层面，软件/文件层面，内容层面）的设备中加载，而决定加载顺序的就是Boot Manager。

**UEFI Images**：符合 UEFI 规范的可执行二进制文件，包含 **UEFI 应用程序（Applications）或 UEFI 驱动（Drivers）**。它们采用 PE/COFF 文件格式——32 位平台使用 PE32，64 位平台（如 x86_64、AArch64、RISC-V64）使用 PE32+。

**UEFI Services**：UEFI Services是UEFI固件（Firmware）本身提供的一套**标准功能接口集合**。允许UEFI应用程序、驱动和UEFI OS Loader调用。分为**Boot Services**和**Runtime Services**。

**UEFI Protocol**：本身是一个接口数据结构（函数指针表），而完整的“协议使用机制”涉及三个关键元素：**GUID、Protocol 接口结构体、以及绑定到 Handle 上的服务实例**。也是UEFI 的“服务扩展机制”，除了 UEFI Services 固件内置的基础服务，想要增加新的接口都必须通过它实现。

**UEFI 系统表**：编写 UEFI 应用程序和UEFI驱动时，需要访问UEFI Protocol。如何定位这些UEFI Protocol呢？所有 UEFI 镜像在启动时都会收到一个指向 UEFI 系统表的指针。开发者通过该表获取 Boot Services，再利用 Boot Services 提供的接口（如 LocateProtocol）来发现和使用所需的 UEFI Protocol。


###  UEFI 构建文件

（1）.dec（Package Declaration File）是包声明文件，定义包的公共接口和资源，供其他模块或包使用。 
（2）.dsc（Platform Description File）是平台描述文件，定义了如何构建一个完整的固件映像，指定了目标平台所需的所有组件（驱动、应用）及其配置（编译选项），是构建特定硬件固件的“总蓝图”。 
（3）.inf(Module Information File)是模块描述文件，描述了如何编译和链接一个独立的模块（如驱动、库或应用），是构建系统处理的最小单元，相当于该模块的“构建说明书”。 
（4）.FDF(Flash Description File)是定义固件映像的最终布局和内容。











##  
在 EDK II 中，每个 gEfiXXXProtocolGuid 是一个 全局常量符号，Protocol/RamDisk.h 只是 声明（extern），其定义位于对应的 .c 或 .inf 模块 中。
 



##  bootflow
启动流程
![image](./assets/boot%20flow.png)




###  DXE（Driver Execution Environment）
通过加载驱动、初始化各种硬件设备、安装协议等操作，逐步建立起Boot Services环境（包括内存服务、协议服务、设备驱动机制等），为系统引导做好准备。关键任务包括：
（1）内存服务初始化：建立内存管理机制，初始化内存映射表，为Boot Services 和Runtime Services提供内存分配与释放支持。
（2）设备枚举与驱动加载：DXE阶段通过PCI总线（或其他总线）进行设备枚举，识别所有的Bus、Bridge和Device。对每个设备，通过设备句柄上的协议（如PciIoProtocol）匹配合适的UEFI驱动模块，加载驱动并初始化设备。
注：设备之间的连接（Connect Controller）动作，主要在BDS阶段触发。

DXE Core = 调度器 / 微内核
DXE Driver = 插件 / 设备建模单元

为符合规范及良好的架构：
任何“改变系统硬件拓扑”的行为（比如注册 RamDisk），
都必须放在某一个具体 DXE Driver 里，
绝不能写进 DxeMain / CoreDispatcher 这种公共调度路径。
##  UEFI Shell

UEFI Shell本质是一个EFI应用程序。

先把 EFI Image 放进内存 → 再把这段内存注册成 RamDisk → 最后通过 Shell/Boot Manager 去加载执行这个 Image。


edk2的shell中，执行应用程序或者镜像的关键：
（1）必须有一个文件系统设备，如fs0,fs1等。只有块设备（如BLK0:），是不够的，需要先将其映射为文件系统设备（fs0:）。
（2）应用程序或者镜像必须符合UEFI要求，比如，有相应的目录结构，格式正确。

img 本质：

*   标准的 UEFI PE/COFF 可执行文件（.efi）

*   结构：
ramdisk.img
└─ EFI
   └─ BOOT
      └─ BOOTRISCV64.EFI 
##  GRUB 

### 制作grubriscv64.efi
克隆grub源码：
```sh
git clone --branch grub-2.14 --single-branch https://git.savannah.gnu.org/git/grub.git

```
安装相关包

```sh
apt install autoconf-archive 
```
设置工具链环境（在build目录中）
```sh
export PATH="/data1/dean/ventana-cross-toolchain-2024.11.15-3/bin:$PATH"
```

GRUB 需要先运行 bootstrap 生成 configure 脚本
```sh
../bootstrap
```
运行源码目录的configure
```sh
mkdir build && cd build
../configure \
  --target=riscv64-unknown-linux-gnu \
  --with-platform=efi \
  --disable-werror \
  --disable-grub-mkfont \
  --disable-device-mapper
```

编译：

```sh
make -j$(nproc)
```

生成grubriscv64.efi：
```sh
./grub-mkimage \
  -O riscv64-efi \
  -o bootriscv64.efi \
  -p /EFI/BOOT \
  -d ./grub-core/ \
  fat ext2 part_gpt part_msdos
```
* -O ： Output format，须和 configure --target=... 时指定的平台匹配，表示生成一个能在 RISC-V 64 位 UEFI 固件上运行的 bootloader。
* -o ： Output file，指定生成的 bootloader 文件名。
* -p ： GRUB 在运行时查找配置文件 grub.cfg 的默认路径。
* -d ： GRUB 模块目录，指定 GRUB 模块的位置。












img镜像结构总览：
```sh
img
 └─ EFI
     └─ BOOT
         ├─ BOOTRISCV64.EFI   # 如果手动执行，grubriscv64.efi 不需改名，自动则需改为“BOOTRISCV64.EFI ”。
         └─ grub.cfg
 └─ boot
     ├─ Image
     ├─ thun-v1.dtb
     └─ initramfs.cpio.bz2
```

##  编写一个简易的 EFI 应用程序

编写一个打印“Hello World”的EFI应用程序。

### 目录结构

```sh
在 edk2 下新建：
edk2/
└── MyApp/
    ├── MyApp.inf
    └── MyApp.c
```
### MyApp.c（最小可运行）

```
#include <Uefi.h>
#include <Library/UefiLib.h>
#include <Library/UefiBootServicesTableLib.h>

EFI_STATUS
EFIAPI
UefiMain (
  IN EFI_HANDLE        ImageHandle,
  IN EFI_SYSTEM_TABLE  *SystemTable
  )
{
  Print(L"Hello from MyApp EFI on RISC-V!\n");
  return EFI_SUCCESS;
}
```

### MyApp.inf（关键）

```
[Defines]
  INF_VERSION    = 0x0001001A
  BASE_NAME      = MyApp
  FILE_GUID      = 12345678-1234-1234-1234-1234567890ab
  MODULE_TYPE    = UEFI_APPLICATION
  VERSION_STRING = 1.0
  ENTRY_POINT    = UefiMain

[Sources]
  MyApp.c

[Packages]
  MdePkg/MdePkg.dec

[LibraryClasses]
  UefiLib
  UefiBootServicesTableLib
```

### 把 MyApp 加入你的 Platform DSC，让 build 知道要编译它。

```
[Components]
  MyApp/MyApp.inf
```

### 修改Platform DSC，原有基础上加入：

```
build -a RISCV64 \
      -b RELEASE \
      -t GCC5 \
      -m MyApp/MyApp.inf \
      -v
check_status "MyApp build"
```

##  制作一个简易的 EFI 镜像

把EFI应用程序打包成镜像文件。

### 创建一个空镜像文件，16MB

```
dd if=/dev/zero of=ramdisk.img bs=1M count=64
```

### 创建 GPT 分区表 + ESP 分区
parted ramdisk.img --script \
mklabel gpt \
mkpart ESP fat32 1MiB 100% \
set 1 esp on

### 设置loop设备，格式化分区为FAT32
LOOP=$(losetup --find --partscan --show ramdisk.img)
mkfs.vfat -F32 ${LOOP}p1
### 挂载镜像
mkdir -p mnt
mount ${LOOP}p1 mnt
### 创建 EFI 标准目录结构
mkdir -p mnt/EFI/BOOT
### 放入 BOOTRISCV64.EFI，假设为“MyApp.efi”
cp MyApp.efi mnt/EFI/BOOT/BOOTRISCV64.EFI
### 同步并卸载
sync
umount mnt
losetup -d $LOOP
rmdir mnt



##  设备树的加载
（1）由固件负责加载 DTB 并通过标准接口传递（U-Boot 用地址，UEFI 用配置表）；
（2）将 DTB 编译进或追加到内核镜像中。
对于在启动命令行中写 dtb=xxx.dtb 让内核自己去加载文件——不是标准做法，在主线内核中不被支持，
尤其在 RISC-V64 和 LoongArch 上明确不可用，在arm中可用是给内核打了补丁实现的。






![image](./assets/分区参数.png)












| 阶段              | 是否使用 HOB  | 用途说明                                      |
| --------------- | --------- | ----------------------------------------- |
| **Pre-memory**  | ✅ 是       | 初始构建 HOB，记录早期信息（如临时内存、启动顺序等）              |
| **Post-memory** | ✅ 是       | 添加系统内存信息、模块加载信息、数据迁移后进行更新等                |
| **DXE 阶段开始后**   | ✅ 读取，不再扩展 | DXE Core 解析并利用 HOB 来完成内存、驱动、服务等初始化工作      |
| **DXE 中期以后**    | ❌ 不再使用    | 系统切换为 Boot Services 机制，HOB 完成历史使命不再参与后续流程 |

