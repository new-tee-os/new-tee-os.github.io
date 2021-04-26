---
title: Keystone 学习笔记
date: 2021-04-24 17:54:06
tags: keystone
---

Keystone 是 RISC-V 架构上的一种 TEE 实现。相对于大多数商业的 TEE 来说，Keystone 的特点是它巧妙地利用了很多 RISC-V 已有的功能来实现其安全性，而不是对现有的指令集进行大量的扩展。因此，现有的 RISC-V CPU 仅需要极小的改动就能够支持 Keystone 的运行。同时，这也使得有 RISC-V 背景的人能够快速适应 Keystone 的思维方式，降低其学习成本。

## Keystone 与 RISC-V

在实现上，Keystone 主要利用了 RISC-V 的以下这些特性：

- RISC-V 通过 **M, S, U** 三个特权模式，将 CPU 上运行的代码分为 **固件，操作系统，用户程序** 三大层级。其中 M 模式运行的通常是固件上的代码，并且拥有对于机器硬件的最高权限。Keystone 对 OpenSBI 这一开源 RISC-V 固件进行了一定的扩展，称为 **security monitor** (**SM**)，以提供高于操作系统的安全管理功能。
- 但是，仅有安全管理功能是不够的，TEE 必须保证 Enclave 与 host OS 之间的可靠 **内存隔离**。而在 RISC-V 上，S 模式拥有对于分页功能的完全控制权，因此在理论上，任何一个 S 模式的程序都可以通过操作分页功能来访问到所有的物理内存。因此，仅靠页表来限制 S 模式的内存访问是不可行的。
- 但除了页表之外，RISC-V 还提供了称为 **物理内存保护**（physical memory protection, PMP）的功能来进一步限制对物理内存的访问。该限制作用于物理内存而非虚拟内存，并且对所有的特权模式（包括 M 模式）均生效。举例说，QEMU 提供 16 个 PMP 寄存器，可以配置 16 个内存区域的访问限制，并且这些寄存器均只能在 M 模式下访问。Keystone 便是利用 PMP 来实现 Enclave 与 host OS 之间的内存隔离。

除了内存隔离，以及与 host OS 之间的交互之外，一个 Keystone Enclave 与一个普通的操作系统 + 用户程序套件并无二致，它们的「操作系统」部分都运行在 S 模式下，并且可以使用 S 模式提供的所有功能，而「用户程序」部分完全由「操作系统」部分管理。

如果想要了解更多的与 Keystone 有关的 RISC-V 背景知识，请参考 Keystone 官方文档中的 [2.1 RISC-V Background](http://docs.keystone-enclave.org/en/latest/Getting-Started/How-Keystone-Works/RISC-V-Background.html) 一节。

## Keystone 架构总览

我们可以把 Keystone 的架构分为 **硬件，M 模式，S 模式，U 模式** 四大部分，详细介绍如下。

- RISC-V 没有内置密码学、信任链等安全功能。因此，在 **硬件** 上，Keystone 需要在 CPU 核心内安装一个 **可信模块**（root of trust），以保护 SM 固件不被恶意修改。（注：此点存疑，因为 Keystone 给 QEMU 的 [补丁](https://github.com/keystone-enclave/keystone/blob/v1.0.0/patches/qemu/qemu-secure-boot.patch) 并没有增加任何硬件功能。）
- 运行在 **M 模式** 上的 SM 是 Keystone 的主体部分，提供内存隔离和 Enclave 管理功能。源代码位于 [sm](https://github.com/keystone-enclave/sm)。
- Enclave 内部也需要有一个运行在 **S 模式** 的部分，对应于一般意义上的操作系统，称为 Keystone Runtime。Keystone 提供的默认 Runtime 称为 **Eyrie**，其源代码位于 [keystone-runtime](https://github.com/keystone-enclave/keystone-runtime)。其代码量较小，但功能丰富，提供 Linux syscall，内存分配，内存加密等功能。
- 如果 host OS 是 Linux，则由于 SM 所提供的功能只有 S 模式下才能够访问，因此 Keystone 需要加载一个 **Linux 内核模块** 来帮助 Enclave 使用 SM 提供的功能，称为 Keystone driver。对应的源代码位于 [linux-keystone-driver](https://github.com/keystone-enclave/linux-keystone-driver)。
- 每个 **Keystone Enclave 可执行文件** 实际上是一个自解压的压缩包，其中包含 **Keystone Runtime**，**host application** 及 **eapp**。程序运行时，会自动解压出所有的文件，然后运行 **host application**，再由 **host application** 连接 **Keystone driver**，加载 **Runtime** 和 **eapp** 并运行。

## Keystone 源代码组织结构

Keystone Enclave 的所有源代码均托管在其 [GitHub 组织](https://github.com/keystone-enclave) 下。由于 Keystone 尚处于测试阶段，因此没有稳定的发行周期，其编译脚本会经常性地通过 `git pull` 直接从 GitHub 上拉取代码。因此在编译前，需要准备好稳定的网络。

Keystone Enclave 有两个主要的入口点：

- 一个是大而全的套件 [keystone](https://github.com/keystone-enclave/keystone)，其中包含了 Keystone 所需的全部组件，包括 Keystone SDK，security monitor，Eyrie runtime，QEMU 等。由于其安装配置步骤较为复杂，并且对环境稳定性有较大要求，该 repo 提供一个预编译的 Docker 镜像帮助配置环境。
- 一个是 Keystone SDK [keystone-sdk](https://github.com/keystone-enclave/keystone-sdk)，仅提供编译 Enclave 所需的部分头文件和库，不提供 QEMU 模拟器等运行环境。适合已经有 Keystone 运行环境的用户使用。

Keystone 的源代码大量使用 git submodules 的方式进行组织，比如上文中提到的 [keystone-sdk](https://github.com/keystone-enclave/keystone-sdk) 实际上是 [keystone](https://github.com/keystone-enclave/keystone) 的一部分。

由于我们没有支持 Keystone 的 RISC-V 开发板，[keystone](https://github.com/keystone-enclave/keystone) 是我们的唯一选择。Keystone 官方提供两种方式来帮助开发者搭建 Keystone 开发环境，分别是 Docker 镜像和手动编译方式。当然，您也可以选择直接从 Keystone 的 Docker 镜像中提取所需的文件。参考我们的 {% post_link extract-keystone-docker-image 另一篇博文 %}。
