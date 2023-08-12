Current tip of the development: [based on ubuntu-mainline 6.4.3](https://github.com/jglathe/linux_ms_dev_kit/tree/jg/ms-dev-kit-2023-v6.4.3)

newest fix: miniDP connector should work better now, it doesn't try to enable a panel (and fail) 

Original from @chenguokai can be found [here](https://github.com/chenguokai/chenguokai/blob/master/tutorial-dev-kit-linux.md)

# Tutorial on Booting Linux on MS Dev Kit 2023

## Introduction

Currently, according to discussions in [linux-surface issue](https://github.com/linux-surface/surface-pro-x/issues/43), it has become possible to boot linux on Microsoft dev kit with key functions working. The latest progress was made by @merckhung who has provided a working kernel tree. However there has been no tutorial that provide a full description on how to make the magic happen. Discussion in the issue by @jglathe has provided a basic working path and my tutorial resembles in many parts. But please note before you begin, neither I nor the code authors would be responsible for any damage on your device or your data. You go at your own risk.

## Overview

The hardware requirements are one MS Dev Kit with stock Windows and a usb key. You will need to install some software on the windows side which will be described later. Following this, you will have a minimal working ubuntu system running on your dev kit.

There are mainly three steps:
1. compile the kernel
2. configure the rootfs
3. flash, boot and enjoy

### Compile the kernel

I have done the following in a ubuntu 20.04 WSL2 environment running on Dev Kit, to avoid annoying cross compiling. You may refer to steps provided by @jglathe for cross compiling guidance.

Firstly, you should install essential tools (`build-essential`, `git`, `bison`, `flex` etc ) for compiling a kernel. If there are any missing tools called in your process. please do some googling and install any missing tools.
Secondly, you should clone merckhung's kernel source to your local machine. The command should be like 

```shell
git clone https://github.com/merckhung/linux_ms_dev_kit
```

The source tree is so huge that you may want to do a [shallow clone](https://git-scm.com/docs/shallow). Currently the default branch in the repo is for Dev Kit, but if it changes in the future, remember to checkout the correct branch:
```shell
git checkout ms-dev-kit-2023-v6.3.0
```


Secondly, you should compile the kernel with the config for Dev Kit. After you enter the kernel source directory, compile with:
```
make devkit_defconfig # this tells the kernel to use Dev Kit default config
make menuconfig       # this allows you to make any custom config change
                      # for ubuntu's initramfs generation requirement, you should enable initramfs support here
                      # press `esc` and choose save after that to save your config
make -j8              # do the actual work, Dev Kit has 8 cores so use 8 threads here
```

After that you will find `Image` file in `arch/arm64/boot` and `sc8280xp-microsoft-dev-kit-2023.dtb` in `arch/arm64/boot/dts
/qcom/`. Copy them out to your Windows partition. You will also find `.config` file right in the top of the kernel source tree, copy it to `/boot` in your WSL system. Root priviledge are required for this copy. You will need to adjust the config file name to meet ubuntu's requirement later. Also, it appears to be necessary to add additional lines for update-initramfs to work:
```
CONFIG_RD_LZ4=y
CONFIG_RD_GZIP=y
```
They happen to be missing, but are required for update-initramfs to know which compression to use.

Finally, except for the kernel image, you will need kernel modules. You can install it on your WSL system by typing 

```sudo make modules_install```

You will find your kernel modules in `/lib/modules/6.3.0+`, you will need to put them in the incoming rootfs.

### Rootfs preparation

In theory, you can use prebuilt raspberry pi ubuntu rootfs image but in my experience, it contains garbage configurations that breaks my experience. Assume that you have an empty rootfs image in ext4 file system (you can always use dd to create one and mkfs to assign a filesystem to it), you can mount it to any place and make a debootstrap on it.
```
sudo debootstrap --arch arm64 focal /mnt http://ports.ubuntu.com/ubuntu-ports # assume you mount rootfs to /mnt and use ubuntu focal
                                                                              # you may replace the final link with your local mirror site
```
You may want to chroot into the new system to setup your root password:

```
sudo cp -r /lib/modules/6.3.0+ /mnt/lib/modules/6.3.0+ # copy kernel modules to rootfs
sudo chroot /mnt
passwd root
# you may also install some softwares in this stage
exit # exit the chroot environment
sudo update-initramfs -c -k 6.3.0+ # you may need to move your kernel config to a proper name in /boot before it works, this step can also be done in WSL
```

After setup, copy the newly generated initrd file in `/boot` (either in WSL2 `/` or `/mnt`) to Windows disk and umount the rootfs and you have got a working ubuntu rootfs.

### Flash, boot and enjoy

You need to move the rootfs image to your Windows disk and flash it to an empty partition. New partitions may be created with Windows's disk manager. You can use winhex(thanks @huxuan0307) to flash the rootfs image to your newly created disk partition. Please always double check you are flashing to the **right partition**.

After your rootfs flashed, you should put your kernel and devicetree file in any place you can find easily. You will need to manually boot the kernel at least the first few times.

As we are lacking ways to install grub to Dev Kit's internal disk now, I would recommend you to create a ubuntu 23.04 (22.04 will NOT work) installation disk with an usb key. You can boot from the usb key and use the grub there to boot the kernel.

When you are greeted by the grub interface, press `c` to enter grub commandline. You will need to type the following command to boot.

```
linux PATH_TO_YOUR_KERNEL_IMAGE clk_ignore_unused efi=novamap earlycon=efifb iommu.strict=0 pd_ignore_unused mitigations=off root=/dev/nvme0n1pX 
# where X is your real rootfs partition number
devicetree PATH_TO_YOUR_DEVICETREE
initrd YOUR_INITRD_FILE
boot
```

You should see ubuntu booting now.

## Appendix

After you have finished this tutorial, not all devices will work as expected, for example the WiFi. However in the problematic RPI rootfs, WiFi works. I suppose that it is mainly because debootstrap will not install `linux-firmware` by default. Currently I have only remote ssh access to Dev Kit and cannot verify myself. You may try it on your own.
Also you may install grub to Dev Kit's internal disk and drop the usb key requirement in the following boots. I have not had time to do it myself as well.
