---
title: "OP-TEE: Môi trường & Gỡ lỗi (Phần 1)"
author: kiennql
date: 2025-06-12 16:34:00 +0700
categories: [optee-labs]
tags: [op-tee, arm trustzone, secure world, qemu, debugging, gdb, cross-compile, arm, repo, gdb-multiarch, embedded linux]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Mục lục
- [1. Mục lục](#1-mục-lục)
- [2. Cấu hình môi trường](#2-cấu-hình-môi-trường)
  - [2.1. Thiết lập môi trường qemu\_v8](#21-thiết-lập-môi-trường-qemu_v8)
  - [2.2. Build với DEBUG mode](#22-build-với-debug-mode)
- [3. Gỡ lỗi với GDB](#3-gỡ-lỗi-với-gdb)
  - [3.1. Kết nối GDB](#31-kết-nối-gdb)
  - [3.2. Symbol files cho các boot stages](#32-symbol-files-cho-các-boot-stages)
  - [3.3. Load symbols theo giai đoạn](#33-load-symbols-theo-giai-đoạn)
- [4. Kiến trúc bảo mật ARM](#4-kiến-trúc-bảo-mật-arm)
  - [4.1. Exception Levels](#41-exception-levels)
  - [4.2. Security States](#42-security-states)
- [5. Quá trình khởi động OP-TEE](#5-quá-trình-khởi-động-op-tee)
  - [5.1. Boot sequence](#51-boot-sequence)
  - [5.2. Debug breakpoints](#52-debug-breakpoints)

Bài viết này hướng dẫn cấu hình môi trường OP-TEE và các kỹ thuật gỡ lỗi cơ bản.

## 2. Cấu hình môi trường

### 2.1. Thiết lập môi trường qemu_v8

Tham khảo: [OP-TEE Prerequisites](https://optee.readthedocs.io/en/latest/building/prerequisites.html)

Tải source code và build:

```bash
repo init -u https://github.com/OP-TEE/manifest.git -m qemu_v8.xml
repo sync -c -j8
cd build
make toolchains
make run
```

Chạy với phiên bản cụ thể:

```bash
make -f qemu_v8.mk run-only
```

### 2.2. Build với DEBUG mode

Tham khảo: [OP-TEE Debug Guide](https://optee.readthedocs.io/en/latest/debug/index.html)

```bash
cd build
make DEBUG=1 -f qemu_v8.mk all
make DEBUG=1 -f qemu_v8.mk run-only
```

Sau khi chạy thành công, hệ thống sẽ hiển thị 2 terminal: một cho Secure World và một cho Normal World.

![Secure World](/assets/img/post/envndebug-optee/secureworld.png)
_Terminal Secure World_

![Normal World](/assets/img/post/envndebug-optee/normalworld.png)
_Terminal Normal World_

## 3. Gỡ lỗi với GDB

### 3.1. Kết nối GDB

Makefile đã cấu hình sẵn tùy chọn `-s -S` để có thể debug cả Normal World và Secure World.

![Make file](/assets/img/post/envndebug-optee/makefile.png)
_Cấu hình Makefile cho debugging_

Thiết lập GDB:

```bash
gdb-multiarch /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl1/bl1.elf
```

Trong GDB console:

```gdb
set architecture aarch64
target remote localhost:1234
```

![Thiết lập GDB](/assets/img/post/envndebug-optee/setupgdb.png)
_Thiết lập kết nối GDB_

Đặt breakpoint và tiếp tục thực thi:

```gdb
break bl1_main
continue
```

![Đặt breakpoint](/assets/img/post/envndebug-optee/breakpoint.png)
_Đặt breakpoint tại bl1_main_

### 3.2. Symbol files cho các boot stages

Danh sách các file symbol cho từng giai đoạn boot:

```bash
# BL1 (Boot ROM)
bl1: /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl1/bl1.elf

# BL2 (Trusted Boot Firmware)  
bl2: /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl2/bl2.elf

# BL31 (EL3 Runtime Software)
bl31: /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl31/bl31.elf

# BL32 (TEE OS)
bl32: /home/kien/workspace/src/optee_os/out/arm/core/tee.elf

# BL33 (UEFI)
bl33: edk2 (không phải u-boot do chạy trên QEMU)
```

### 3.3. Load symbols theo giai đoạn

Khi debug, cần load symbol file tương ứng với giai đoạn hiện tại:

```gdb
file /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl1/bl1.elf
file /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl2/bl2.elf
file /home/kien/workspace/src/trusted-firmware-a/build/qemu/debug/bl31/bl31.elf
file /home/kien/workspace/src/optee_os/out/arm/core/tee.elf
```

## 4. Kiến trúc bảo mật ARM

### 4.1. Exception Levels

![ARM v8-A](/assets/img/post/envndebug-optee/armv8a.png)
_Kiến trúc ARM v8-A với các Exception Levels_

ARM v8-A được chia thành 4 Exception Levels:
- **EL0**: Ứng dụng người dùng (Client Applications, Trusted Applications)
- **EL1**: Hệ điều hành (Linux, OP-TEE OS)
- **EL2**: Hypervisor
- **EL3**: Secure Monitor (ARM Trusted Firmware)

### 4.2. Security States

Hệ thống được chia thành hai "thế giới" bảo mật:
- **Non-secure World**: Linux OS và Client Applications (CA)
- **Secure World**: OP-TEE OS và Trusted Applications (TA)

![Security States](/assets/img/post/envndebug-optee/securitystates.png)
_Hai trạng thái bảo mật trong ARM TrustZone_

## 5. Quá trình khởi động OP-TEE

### 5.1. Boot sequence

![Quá trình khởi động](/assets/img/post/envndebug-optee/bootsequence.png)
_Sơ đồ quá trình khởi động OP-TEE_

Chi tiết quá trình khởi động:

```
bl31_entrypoint (trusted-firmware-a/bl31/aarch64/bl31_entrypoint.S)
└── bl31_main (trusted-firmware-a/bl31/bl31_main.c)
    ├── runtime_svc_init (trusted-firmware-a/common/runtime_svc.c)
    │   └── opteed_setup (trusted-firmware-a/services/spd/opteed/opteed_main.c)
    │       ├── bl31_plat_get_next_image_ep_info(SECURE)
    │       ├── opteed_init_optee_ep_state
    │       └── bl31_register_bl32_init(&opteed_init)
    ├── bl32_init // opteed_init() đã được đăng ký
    │   └── opteed_synchronous_sp_entry(optee_ctx)
    │       └── opteed_enter_sp(&optee_ctx->c_rt_ctx) // ERET → S-EL1 → OP-TEE start
    └── bl31_prepare_next_image_entry // chuyển sang UEFI
```

### 5.2. Debug breakpoints

Để debug hiệu quả, cần đặt breakpoint theo từng giai đoạn boot và load symbol file tương ứng trước khi thực hiện debug.

**Chúc mừng!** Bạn đã thiết lập thành công môi trường OP-TEE và có thể bắt đầu debug các thành phần trong Secure World.
