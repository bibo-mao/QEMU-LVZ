# QEMU-LVZ

本项目为 LoongArch64 架构的 QEMU TCG 虚拟机提供虚拟化扩展（LVZ）支持。目前已基本实现所有指令、CSR及架构支持，可以正常运行启动。

## 快速开始

### 准备
首先需要准备 QEMU 的依赖环境，具体参见：
- https://wiki.qemu.org/Hosts/Linux
- https://wiki.qemu.org/Hosts/Mac
- https://wiki.qemu.org/Hosts/W32

### 下载源码
```bash
$ git clone https://github.com/Hengyu-Yu/QEMU-LVZ.git
```

### 编译

在项目根目录依次执行：
```bash
$ ./configure --target-list=loongarch64-softmmu --disable-docs
$ make -j
```

### 测试
这里我们提供了测试用的镜像与固件，下载链接： https://krgm.moe/static/lvztest.tar.gz

将系统镜像与固件下载到项目根目录后，可以执行以下命令启动外侧tcg虚拟机：
```bash
$ ./build/qemu-system-loongarch64 \
$ 	-hda lvztest.img \
$ 	-bios edk2-loongarch64-code.fd \
$ 	-smp 8 \
$ 	-m 8G \
$ 	-nographic
```

启动后等待一段时间进入登录界面，输入用户名 root，密码为空（直接回车），进入shell。

在虚拟机中再次执行 ./qemu.sh 进入内侧 KVM 虚拟机，等待系统启动（默认启动过程没有串口显示）。

### 💡 MacOS 环境构建提示
如果 Mac 中构建 QEMU 时发生编译链接错误 archive member '/' not a mach-o file ，则需在编译前，将 binutils 从环境变量中临时剔除，或者直接重置为默认（这是 MacOS 上编译 C/C++ 项目常见的一个问题，非本项目Bug），可以通过下面的命令：
```bash
$ export PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin
```

## 设计

### CPU虚拟化

在 CPULoongArchState 结构体中加入 Guest 变量，以确定 Host/Guest 模式，为两个模式提供了两套不同的 TLB、CSR 寄存器与定时器。Host 模式下通过设置 CSR.GSTAT.PGM 为 1 后执行 ertn，进入 Guest 模式；Guest模式下触发 HVCL、GSPR 以及与 Host 地址翻译相关的例外，或 Host 触发中断，回到 Host 模式。

### 内存虚拟化

LoongArch 继承了 MIPS 软件处理页表的设计，硬件通过 TLB 完成访存。GVA-\>HPA 的完整地址转换模式如下：

1. 在 Guest TLB 中查找 GVA-\>GPA 的映射。若未找到，直接在 Guest 触发例外，不退回 Host。
2. 找到 Guest TLB 后，在 Host TLB 中查找 GPA-\>HPA 的映射。若未找到，退回Host并触发TLB重填例外。
3. Host 在页表中查找 GPA-\>HPA 项并填入 TLB，若页表不存在该项，则会填入 V=0 项。
4. 再次查找 TLB，若找到 V=0 项，则退回 Host 并触发页无效例外。
5. 进入 Guest 模式前，Host 的一般例外入口已被 KVM 设置，因此后续由 KVM 处理 GPA->HPA 映射，更新页表。


### 定时器虚拟化

为 Guest 模式单独实现了一个定时器，在进入 Guest 模式时，若 TCFG 已配置则开启定时器，退出 Gust 模式后无条件停止。定时器到期则触发 Guest 中断。

Host 模式期间，由kvm通过软件模拟定时器计时，进入 Guest 时将更新的时间写回 Guest CSR.TVAL。若在此期间定时器到期，则将 ESTAT 中定时器中断位置 1，并触发中断。

### 中断虚拟化

LoongArch64 架构提供了多种将 Host 中断映射到 Guest 的方式：

- CSR.GINTCTL.HWIC 被设置时，外部硬件的中断由 Host 处理，Host向Guest的CSR.ESTAT 对应中断位置 1，则 Host的CSR.ESTAT 对应位自动归 0。
- CSR.GINTCTL.HWIP 被设置时，外部硬件的中断直接映射到Guest。

Guest 模式下，Host 依旧响应自己的中断，不会长时间占据 CPU。 kvm\_handle\_exit 函数会在将中断处理入口换成常规入口后，开启中断，由 Host 处理自己的中断。

### 杂项

MMIO 访问与上述内存访问流程大致相同，不同点为 KVM  直接模拟  MMIO 操作，不更新页表。

执行 IOCSR、CPUCFG、IDLE 等指令，直接触发 GSPR 例外，由 KVM 处理。

### 与龙芯架构参考手册卷三的设计差异

1. 卷三中提到 Host 与 Guest 共享 TLB ，但未阐述如何避免 TLB 项冲突，因此目前采用 Guest、Host TLB 分开实现的方法。
2. invtlb 在文档中共有 15 个操作值，目前尚未对所有操作值进行实现，对 0x10 - 0x16 值采用了对指定 GID 项的 TLB 项全部无效化的实现，理论上不应影响运行的正确性。
