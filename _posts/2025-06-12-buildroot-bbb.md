---
title: Tạo hệ thống Linux tối thiểu cho BeagleBone Black với Buildroot (Phần 5)
author: kiennql
date: 2025-06-12 08:47:00 +0700
categories: [bbb-labs]
tags: [beaglebone black, embedded linux, buildroot, build system, cross-compile, arm]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Build System là gì?

Build system là công cụ tự động hóa việc biên dịch source code thành binary files. Buildroot tự động build:
- Linux kernel
- Bootloader (U-Boot)
- Root filesystem
- Các package và library cần thiết

Điều này giúp tiết kiệm thời gian và giảm lỗi so với việc build thủ công.

## 2. Cài đặt Buildroot

Clone Buildroot repository:

```bash
git clone https://git.buildroot.net/buildroot
cd buildroot
git checkout 2022.11
```

## 3. Cấu hình Buildroot

Mở menu cấu hình:

```bash
make menuconfig
```

![Menuconfig](/assets/img/post/buildroot-bbb/image.png)
_Menuconfig_

**Target Options:**
- Target Architecture: **ARM (little endian)**
- Target Architecture Variant: **cortex-A8**

**Toolchain:**
- Toolchain type: **External toolchain**
- Toolchain: **ARM 2021.07**

**System Configuration:**
- System hostname: Đặt tên tùy ý (vd: `kaibeaglebone`)
- System banner: Thông điệp hiển thị khi boot (vd: `Kai's Kernel !!!`)
- Enable root login with password và đặt mật khẩu

**Kernel:**
- Enable **Linux Kernel**
- Defconfig name: **omap2plus**
- Kernel binary format: **zImage**
- Device Tree Source file: **am335x-boneblack**
- Enable **Needs host OpenSSL**

**Target Packages:**
- Enable **BusyBox**
- Optional: Enable **dropbear** (SSH client)
- Optional: Enable **gdb**, **gdbserver**, và **valgrind** trong Debugging

**Bootloaders:**
- Enable **U-Boot**
- U-Boot configuration: **am335x_evm**
- Build system: **Kconfig**
- U-Boot Version: **2022.04**
- U-Boot binary format: **u-boot.img**
- Enable **Install U-Boot SPL binary image**
- U-boot SPL binary image name: **MLO**

## 4. Build hệ thống

Bắt đầu build (có thể mất 30-60 phút):

```bash
make -j$(nproc)
```

Sau khi hoàn thành, các file output sẽ có trong thư mục `output/images/`:
- `MLO`
- `u-boot.img`
- `zImage`
- `am335x-boneblack.dtb`
- `rootfs.tar`

## 5. Chuẩn bị thẻ SD

Tạo 2 phân vùng trên thẻ SD bằng [**GParted**](https://gparted.org/):
- Phân vùng 1: FAT32, 128MB, có boot flag
- Phân vùng 2: ext4, phần còn lại

![Phân vùng SD bằng GParted](/assets/img/post/buildroot-bbb/image_sd_gparted.png)
_Phân vùng đã tạo thành công: /dev/sdX1 (FAT32, boot) và /dev/sdX2 (ext4)_

Copy bootloader và kernel:

```bash
cp output/images/MLO output/images/u-boot.img output/images/zImage output/images/am335x-boneblack.dtb /media/$USER/boot/
```

Tạo file cấu hình U-Boot:

```bash
mkdir /media/$USER/boot/extlinux
cat > /media/$USER/boot/extlinux/extlinux.conf << 'EOF'
label buildroot
        kernel /zImage
        devicetree /am335x-boneblack.dtb
        append console=ttyO0,115200 root=/dev/mmcblk0p2 rootwait
EOF
```

Giải nén root filesystem:

```bash
sudo tar -C /media/$USER/rootfs/ -xf output/images/rootfs.tar
```

## 6. Boot hệ thống

Kết nối serial và khởi động minicom:

```bash
minicom -D /dev/ttyUSB0 -b 115200
```

Chèn thẻ SD vào BeagleBone, giữ nút Reset button và cấp nguồn để boot từ SD card.

Bạn sẽ thấy U-Boot tự động load kernel và boot vào Linux với banner đã cấu hình.

Login bằng user `root` và mật khẩu đã đặt.

**Chúc mừng!** Bạn đã thành công tạo hệ thống Linux hoàn chỉnh với Buildroot cho BeagleBone Black.

Buildroot đã tự động hóa toàn bộ quá trình mà chúng ta đã làm thủ công trong 4 phần trước, từ toolchain đến root filesystem.
