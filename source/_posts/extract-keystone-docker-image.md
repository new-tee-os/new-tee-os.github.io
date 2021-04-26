---
title: 从 Keystone 的 Docker 镜像中提取 QEMU 并运行
date: 2021-04-26 18:04:42
tags: keystone
---

Keystone 的一次完整构建大约需要消耗 4 GB 的硬盘空间，但我们所需要的很可能只是其中的一小部分（比如我们可能只需要一个 QEMU 作为运行和测试环境）。由于 Keystone 的 Docker 镜像是已经预先编译好的，我们可以尝试从 Docker Hub 上直接下载 Keystone 的镜像，并提取我们所需要的文件。

注意，Docker 镜像中的 QEMU 是动态链接的，因此系统上需要预先安装有所需的库，否则 QEMU 将无法正常启动。笔者总结的依赖列表如下：

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

（注：以上这些文件自上而下分别是：2 个 Keystone SM 固件，2 个 Linux 内核，Linux 根文件系统，QEMU RISC-V 可执行文件，以及 Keystone 的一些测试脚本。最后一项是可选的。）

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
