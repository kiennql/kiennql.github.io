---
title: Biên dịch và porting Linux Kernel cho BeagleBone Black (Phần 3)
author: kiennql
date: 2025-06-10 23:23:00 +0700
categories: [bbb-labs]
tags: [beaglebone black, embedded linux, linux kernel, cross-compile, arm]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Tải source code

Clone Linux repository:

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux
```

Repository này khoảng 2.5GB, hãy kiên nhẫn chờ đợi.

Thêm stable repository và fetch:

```bash
cd linux
git remote add stable https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux
git fetch stable
```

Checkout phiên bản stable mới nhất:

```bash
git checkout remotes/stable/linux-6.1.y
```

## 2. Cross-compile kernel

Thiết lập biến môi trường:

```bash
export ARCH=arm
export CROSS_COMPILE=arm-linux-
```

Cấu hình kernel cho BeagleBone Black:

```bash
make omap2plus_defconfig  # hoặc multi_v7_defconfig
```

Mở menu cấu hình và tắt `CONFIG_GCC_PLUGINS`:

```bash
make menuconfig
```
Trong menu, tìm và disable `CONFIG_GCC_PLUGINS` (dùng `/` để search).

![Kernel Configuration Menu](/assets/img/post/kernel-bbb/b7746e2c-060d-42df-936f-a04cec9f6a85.jpg)
_Menu cấu hình kernel Linux với các tùy chọn build_

Biên dịch kernel:

```bash
make -j$(nproc)
```

Quá trình này sẽ mất một thời gian tùy thuộc vào CPU.

![Kernel Build Complete](/assets/img/post/kernel-bbb/8fe40e82-0b8c-4262-9d8b-47a6da333ac8.jpg)
_Quá trình build kernel hoàn tất thành công_

Copy các file cần thiết vào thẻ SD:

```bash
cp arch/arm/boot/zImage /media/$USER/boot/
cp arch/arm/boot/dts/am335x-boneblack.dtb /media/$USER/boot/
```

## 3. Boot kernel

Khởi động minicom:

```bash
minicom -D /dev/ttyUSB0 -b 115200
```

Boot BeagleBone từ SD card (giữ nút Reset button khi cấp nguồn).

Tại prompt U-Boot `=>`, thiết lập boot arguments:

```
=> setenv bootargs console=ttyS0,115200n8
=> saveenv
```

Load kernel và device tree vào memory:

```
=> fatload mmc 0:1 0x80200000 zImage
=> fatload mmc 0:1 0x80f00000 am335x-boneblack.dtb
```

Boot kernel:

```
=> bootz 0x80200000 - 0x80f00000
```

Bạn sẽ thấy kernel boot và cuối cùng gặp kernel panic về việc không tìm thấy root filesystem. Điều này là bình thường vì chúng ta chưa tạo root filesystem.

**Tự động hóa quá trình boot:**

```
=> setenv bootcmd 'fatload mmc 0:1 0x80200000 zImage; fatload mmc 0:1 0x80f00000 am335x-boneblack.dtb; bootz 0x80200000 - 0x80f00000'
=> saveenv
```

**Chúc mừng!** Bạn đã thành công biên dịch và boot Linux kernel trên BeagleBone Black.
