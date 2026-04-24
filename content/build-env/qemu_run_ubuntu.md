+++
date = '2026-04-24T08:25:28+08:00'
draft = false 
title = 'QEMU安装ubuntu虚拟机'
+++



## 背景

最近想跟下QEMU和KVM代码执行，由于物理机安装的是ubuntu22.04系统，如果直接在物理机上安装自己编译的linux内核（含kvm），很容易导致主机的崩溃，同时也不大好跟踪kvm代码的执行；于是乎想到以下解决方案：首先用主机qemu安装一个ubuntu虚拟机，然后编译一个自己的linux内核镜像（带kvm内核模块），当qemu启动ubuntu虚拟机时指定内核为自己编译的内核，这样当ubuntu虚拟机中的qemu程序运行时就可以跟踪内核中的kvm代码了，如下图所示
![background_1](/images/qemu_run_ubuntu/background_1.png)

## 预安装包

```bash
# 安装OVMF固件以支持UEFI引导（seabios可能不支持最新的ubuntu发行版）
sudo apt install ovmf
# 由于qemu在安装ubuntu的过程中会修改OVMF_VARS_4M.fd文件，而修改该文件需要root权限，所以拷贝一份该文件到当前目录以绕过权限的问题
cp /usr/share/OVMF/OVMF_VARS_4M.fd .
# 为了让mkinitramfs命令在执行时把lvm2工具打包进initrd.img镜像，以便initramfs可以识别在LVM逻辑卷上的ubuntu根文件系统，进而挂载ubuntu根文件系统，我们需要在主机上安装lvm2工具包（由于我们选择安装ubuntu22.04-server，该发行版安装时默认根文件系统在LVM上，当然你也可以在安装时不支持LVM，这样就可以跳过主机lvm2工具包的安装）
sudo apt install lvm2
# 在编译qemu时，为了让qemu支持-net user选项，主机需要安装libslirp-dev，qemu中-net user是qemu内部实现的一个NAT网络环境（如果你的qemu是通过apt安装的而不是自己编译的可以忽略这个步骤）
sudo apt install libslirp-dev
```

## 编译qemu（apt安装跳过）

```bash
cd /path/to/qemu
mkdir build
cd build
# -target-list支持的架构:
# aarch64-linux-user aarch64_be-linux-user alpha-linux-user arm-linux-user armeb-linux-user hexagon-linux-user hppa-linux-user i386-linux-user loongarch64-linux-user m68k-linux-user microblaze-linux-user microblazeel-linux-user mips-linux-user mips64-linux-user mips64el-linux-user mipsel-linux-user mipsn32-linux-user mipsn32el-linux-user or1k-linux-user ppc-linux-user ppc64-linux-user ppc64le-linux-user riscv32-linux-user riscv64-linux-user s390x-linux-user sh4-linux-user sh4eb-linux-user sparc-linux-user sparc32plus-linux-user sparc64-linux-user x86_64-linux-user xtensa-linux-user xtensaeb-linux-user aarch64-softmmu alpha-softmmu arm-softmmu avr-softmmu hppa-softmmu i386-softmmu loongarch64-softmmu m68k-softmmu microblaze-softmmu microblazeel-softmmu mips-softmmu mips64-softmmu mips64el-softmmu mipsel-softmmu or1k-softmmu ppc-softmmu ppc64-softmmu riscv32-softmmu riscv64-softmmu rx-softmmu s390x-softmmu sh4-softmmu sh4eb-softmmu sparc-softmmu sparc64-softmmu tricore-softmmu x86_64-softmmu xtensa-softmmu xtensaeb-softmmu
# 我们只编译x86_64架构，如下
../configure --target-list=x86_64-softmmu,x86_64-linux-user
make -j
# 把/path/to/qemu/build添加到可执行文件默认搜索路径
export PATH=/path/to/qemu/build:$PATH
```

## qemu安装并启动ubuntu

- 选择ubuntu镜像

为了不需要图形化组件的支持，我们选择ubuntu22.04 server版

```bash
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
```

- 创建qcow2镜像

```bash
# 50G是创建的磁盘大小
qemu-img create -f qcow2 ubuntu-22.04.5-server.qcow2 50G
```

- qemu安装ubuntu
    - 运行安装命令
    ```bash
    qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 16G -smp 4 -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd -drive if=pflash,format=raw,file=./OVMF_VARS_4M.fd -cdrom ubuntu-22.04.5-live-server-amd64.iso -boot order=d -hda ubuntu-22.04.5-server.qcow2 -netdev user,id=net0,hostfwd=tcp::2222-:22 -device e1000,netdev=net0 -serial mon:stdio -display none
    ```
    - 将光标定位到Try or Install Ubuntu Server，按'e'键修改启动命令，如下图所示
    ![qemu_install_ubuntu_1](/images/qemu_run_ubuntu/qemu_install_ubuntu_1.png)
    - 添加console=ttyS0到linux那行最末尾，如下图所示
    ![qemu_install_ubuntu_2](/images/qemu_run_ubuntu/qemu_install_ubuntu_2.png)
    - 按Ctrl+X启动安装，安装过程选择默认选项即可完成
    - 安装成功后重启，如下图所示
    ![qemu_install_ubuntu_3](/images/qemu_run_ubuntu/qemu_install_ubuntu_3.png)

- qemu启动ubuntu
```bash
# 安装完成后，我们可以去掉-cdrom -boot参数启动ubuntu
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 16G -smp 4 -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd -drive if=pflash,format=raw,file=./OVMF_VARS_4M.fd -hda ubuntu-22.04.5-server.qcow2 -netdev user,id=net0,hostfwd=tcp::2222-:22 -device e1000,netdev=net0 -serial mon:stdio -display none
```


## 用自己编译的内核启动ubuntu

- 编译linux内核

```bash
cd /path/to/linux
mkdir build
cp arch/x86/configs/x86_64_defconfig build/.config
make O=build ARCH=x86_64 menuconfig
    ---> Virtualization
```

在Virtualization配置中添加内核的kvm支持，注意需要直接编译到内核，而不是编译成内核模块，如下图所示
![build_kernel_1](/images/qemu_run_ubuntu/build_kernel_1.png)

```bash
# 配置完成后编译内核
make O=build ARCH=x86_64 -j20
# 生成的内核镜像路径
ls /path/to/linux/build/arch/x86/boot/bzImage
# 生成initrd.img镜像
cd build
mkinitramfs -o initrd.img
# 生成的initrd.img镜像路径
ls /path/to/linux/build/initrd.img
```

- 用编译的内核启动ubuntu
```bash
qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 16G -smp 4 -kernel /path/to/linux/build/arch/x86/boot/bzImage -initrd /path/to/linux/build/initrd.img -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd -drive if=pflash,format=raw,file=./OVMF_VARS_4M.fd -append "root=/dev/mapper/ubuntu--vg-ubuntu--lv console=ttyS0" -hda ubuntu-22.04.5-server.qcow2 -netdev user,id=net0,hostfwd=tcp::2222-:22 -device e1000,netdev=net0 -serial mon:stdio -display none
```
