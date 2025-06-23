---
title: Tạo Root Filesystem cho BeagleBone Black (Phần 4)
author: kiennql
date: 2025-06-10 23:39:00 +0700
categories: [bbb-labs]
tags: [beaglebone black, embedded linux, root filesystem, busybox, cross-compile, arm]
description: "Tạo root filesystem cho BeagleBone Black từ đầu với BusyBox, cấu hình init system và hoàn thiện hệ thống Linux tối thiểu."
math: true
mermaid: true
render_with_liquid: false
---

## 1. Tạo Root Filesystem từ đầu

Tạo thư mục staging với cấu trúc filesystem chuẩn:

```bash
mkdir rootfs && cd rootfs
mkdir bin dev etc home lib proc sbin sys tmp usr var
mkdir usr/bin usr/lib usr/sbin var/log
```

Kiểm tra cấu trúc với `tree -d`:

```
.
├── bin
├── dev
├── etc
├── home
├── lib
├── proc
├── sbin
├── sys
├── tmp
├── usr
│   ├── bin
│   ├── lib
│   └── sbin
└── var
    └── log
```

## 2. Biên dịch BusyBox

Clone BusyBox repository:

```bash
git clone git://busybox.net/busybox.git
cd busybox
git checkout 1_35_0
```

Cấu hình và build:

```bash
make distclean
make defconfig
export ARCH=arm
export CROSS_COMPILE=arm-linux-
```

Cấu hình BusyBox:

```bash
make menuconfig
```

Trong menu config:
- **Settings** → **Destination path for 'make install'**: Đặt đường dẫn đến thư mục rootfs
- **Settings** → **Build static binary**: Bật tùy chọn này
- **Networking Utilities** → **Enable IPv6 Support**: Tắt nếu gặp lỗi compile

Build và install:

```bash
make install
```

## 3. Kiểm tra hệ thống

Copy rootfs vào thẻ SD:

```bash
sudo cp -r rootfs/* /media/$USER/rootfs/
```

Boot BeagleBone và tại U-Boot prompt, thiết lập boot arguments:

```
=> setenv bootargs console=ttyO0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait init=/bin/sh
=> boot
```

Bạn sẽ thấy hệ thống boot thành công và có shell prompt:

```
/ # ls
bin      etc      lib      proc     sys      usr
dev      home     linuxrc  sbin     tmp      var
```

## 4. Cấu hình Init System

Tạo các file cấu hình init trong thư mục staging:

```bash
cd rootfs/etc
mkdir init.d
touch init.d/rcS
touch inittab
```

Tạo file `inittab`:

```bash
cat > inittab << 'EOF'
# /etc/inittab
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
EOF
```

File `init.d/rcS` có thể để trống hoặc thêm các lệnh khởi tạo:

```bash
cat > init.d/rcS << 'EOF'
#!/bin/sh
# System initialization script
echo "Starting system..."
EOF

chmod +x init.d/rcS
```

Copy các file này vào thẻ SD và reboot:

```bash
sudo cp etc/inittab /media/$USER/rootfs/etc/
sudo cp etc/init.d/rcS /media/$USER/rootfs/etc/init.d/
```

Thay đổi boot arguments để sử dụng init system:

```
=> setenv bootargs console=ttyO0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait
=> boot
```

**Chúc mừng!** Bạn đã thành công tạo một hệ thống Linux tối thiểu hoàn chỉnh trên BeagleBone Black.

Hệ thống giờ đây có đầy đủ các tiện ích cơ bản từ BusyBox và có thể chạy các lệnh Linux quen thuộc như `ls`, `cat`, `ps`, `mount`, v.v.
