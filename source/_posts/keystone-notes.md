---
title: Keystone 学习笔记
date: 2021-04-24 17:54:06
tags: keystone
---

Keystone 是 RISC-V 架构上的一种 TEE 实现。相对于大多数商业的 TEE，Keystone 的特点是它巧妙地利用了很多 RISC-V 已有的功能来实现其安全性，而不是选择在现有指令集的基础上进行大量扩展。因此，现有的 RISC-V CPU 仅需要极小的改动就能够支持 Keystone。同时，这也使得有 RISC-V 背景的人能够快速适应 Keystone 的思维方式，降低其学习成本。

## Keystone 与 RISC-V

在实现上，Keystone 主要利用了 RISC-V 的以下这些特性：

- RISC-V 通过 M, S, U 三个特权模式，将 CPU 上运行的代码分为 固件，操作系统，用户程序 三大层级。其中 M 模式运行的通常是固件上的代码，并且拥有对于机器硬件的最高权限。Keystone 对 OpenSBI 这一开源 RISC-V 固件进行了一定的扩展，称为 **security monitor** (**SM**)，以提供高于操作系统的安全管理功能。
- 但是，仅有安全管理功能是不够的，TEE 必须保证 Enclave 与 host OS 之间的可靠 **内存隔离**。而在 RISC-V 上，S 模式拥有对于页表的完全控制权，因此在理论上，任何一个 S 模式的程序都可以通过操作分页功能来访问到所有的物理内存。因此，仅靠页表来限制 S 模式的内存访问是不可行的。
- 但除了页表之外，RISC-V 还提供了称为 **物理内存保护**（physical memory protection, PMP）的功能来进一步限制对物理内存的访问。通常 PMP 提供 8 个寄存器，可以设置 8 个内存区域的访问权限，并且这些寄存器均只能在 M 模式下访问。Keystone 便是利用 PMP 来实现 Enclave 对 host OS 的内存隔离。

除了内存隔离和与 host OS 的有限交互功能之外，一个 Keystone Enclave 与一个普通的操作系统套件并无二致，它们都运行在 S 模式下，并且可以使用 S 模式提供的所有功能。

## Keystone 架构总览

我们可以把 Keystone 的架构分为 硬件，M 模式，S 模式 三大部分，详细介绍如下。

【TODO……】

## Keystone 源代码组织结构

Keystone Enclave 的所有源代码均托管在其 [GitHub 组织](https://github.com/keystone-enclave) 下。由于 Keystone 尚处于测试阶段，因此没有稳定的发行周期，其编译脚本会经常性地通过 `git pull` 直接从 GitHub 上拉取代码。因此在编译前，需要准备好稳定的网络。

Keystone Enclave 有两个主要的入口点：

- 一个是大而全的套件 [keystone](https://github.com/keystone-enclave/keystone)，其中包含了 Keystone 所需的全部组件，包括 Keystone SDK，security monitor，Eyrie runtime，QEMU 等。由于其安装配置步骤较为复杂，并且对环境稳定性有较大要求，该 repo 提供一个预编译的 Docker 镜像帮助配置环境。
- 一个是 Keystone SDK [keystone-sdk](https://github.com/keystone-enclave/keystone-sdk)，仅提供编译 Enclave 所需的部分头文件和库，不提供 QEMU 模拟器等运行环境。适合已经有 Keystone 运行环境的用户使用。

Keystone 的源代码大量使用 git submodules 的方式进行组织，比如上文中提到的 [keystone-sdk](https://github.com/keystone-enclave/keystone-sdk) 实际上是 [keystone](https://github.com/keystone-enclave/keystone) 的一部分。

由于我们没有支持 Keystone 的 RISC-V 开发板，[keystone](https://github.com/keystone-enclave/keystone) 是我们的唯一选择。Keystone 官方提供两种方式来帮助开发者搭建 Keystone 开发环境，分别是 Docker 镜像和手动编译方式。当然，您也可以选择直接从 Keystone 的 Docker 镜像中提取所需的文件，见下文。

## 从 Keystone 的 Docker 镜像中提取有用的文件

完整的 Keystone 镜像可能会过大，有时我们可能只需要其中的一小部分（比如我们可能只需要一个 QEMU 作为运行环境）。由于 Keystone 的 Docker 镜像是已经预先编译好的，我们可以尝试从 Docker Hub 上直接下载 Keystone 的镜像，并提取我们所需要的文件。

注意，该镜像中的 QEMU 是动态链接的，因此系统上需要安装有所需的库，否则 QEMU 将无法正常启动。笔者总结的依赖列表如下：

```
glib2 zlib pixman util-linux-libs gcc-libs libffi pcre
```

如果以上的依赖列表不能解决您的问题，您也可以尝试直接从 Docker 镜像中提取所需的库文件。

从 Docker Hub 下载镜像可以参考这篇问答：[Downloading Docker Images from Docker Hub without using Docker](https://devops.stackexchange.com/questions/2731/downloading-docker-images-from-docker-hub-without-using-docker)，参见题主的 **UPDATE 2** 部分。以下列出所需使用的 URL 以供参考。

- 认证：`https://auth.docker.io/token?service=registry.docker.io&scope=repository:keystoneenclaveorg/keystone:pull`
- Pull image manifest: `https://registry-1.docker.io/v2/keystoneenclaveorg/keystone/manifests/master`（注意分支名不是 `latest`）
- 下载镜像：`https://registry-1.docker.io/v2/keystoneenclaveorg/keystone/blobs/sha256:$SHA256`

在「Pull an image manifest」步骤，Docker Hub 会列出 10 多个镜像，如下图：

{% asset_img keystone-docker-layers.png %}

我们需要的是顺数第二个，其中包含已经完整编译好的 Keystone 套件。在本篇文章写作时，镜像 SHA256 为 `799863999c96decb76c0946b91375fcc1c528be466548cf8c55389592031a397`，大小约为 3.56 GB。

镜像格式为 tar.gz，可以通过 curl 参数 `-o keystone.tar.gz` 将其直接保存下来。（小提示：如果因为网络环境不稳定导致下载中断，可以添加 `-C -` 参数恢复下载。）

---

镜像下载完成之后，即可从中提取我们所需要的文件。为了运行完整的 Keystone 测试套件，我们需要的文件列表如下：

```
keystone/build/bootrom.build/bootrom.bin
keystone/build/bootrom.build/bootrom.elf
keystone/build/sm.build/platform/generic/firmware/fw_payload.bin
keystone/build/sm.build/platform/generic/firmware/fw_payload.elf
keystone/build/buildroot.build/images/rootfs.ext2
keystone/qemu/riscv64-softmmu/qemu-system-riscv64
keystone/build/scripts/*
```

（注：以上这些文件自上而下分别是：2 个 Keystone SM 固件，2 个 Linux 内核，Linux 根文件系统，QEMU RISC-V，以及 Keystone 的一些测试脚本。最后一项是可选的。）

为了方便，可以将以上内容保存成文件 `requirements.txt`。然后，通过以下命令来解压所需的文件：

```sh
cat requirements.txt | xargs tar xzf keystone.tar.gz --wildcards
```

解压后将生成 `keystone` 目录。通过以下命令来启动 QEMU：

```
keystone/qemu/riscv64-softmmu/qemu-system-riscv64 -m 2G -nographic \
    -machine virt -bios keystone/build/bootrom.build/bootrom.bin \
    -kernel keystone/build/sm.build/platform/generic/firmware/fw_payload.elf \
    -append "console=ttyS0 ro root=/dev/vda" \
    -drive file=keystone/build/buildroot.build/images/rootfs.ext2,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    -netdev user,id=net0,net=192.168.100.1/24,dhcpstart=192.168.100.128,hostfwd=tcp::8022-:22 \
    -device virtio-net-device,netdev=net0 -device virtio-rng-pci -smp 1
```

其中，`hostfwd=tcp::8022-:22` 会使得 QEMU 在 8022 端口上转发 SSH 连接。QEMU 内的系统成功启动后，应该能看到如下提示：

```
Welcome to Buildroot
buildroot login:
```

系统的登录用户名为 `root`，密码为 `sifive`。成功登录之后，执行以下命令来测试 Keystone 是否正常工作：

```sh
insmod keystone-driver.ko
./tests.ke
```

运行 `tests.ke` 时，应该能看到如下输出：

```
# ./tests.ke
Verifying archive integrity... All good.
Uncompressing Keystone Enclave Package
testing stack
testing loop
testing malloc
testing long-nop
testing fibonacci
testing fib-bench
testing attestation
Attestation report SIGNATURE is valid
testing untrusted
Enclave said: hello world!
Enclave said: 2nd hello world!
Enclave said value: 13
Enclave said value: 20
testing data-sealing
Enclave said: Sealing key derivation successful!
```

完成测试后，可以通过 `poweroff` 命令关闭虚拟机并退出 QEMU。
