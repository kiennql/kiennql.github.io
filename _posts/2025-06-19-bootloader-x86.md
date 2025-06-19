---
title: Viết Bootloader đầu tiên cho kiến trúc x86
author: kiennql
date: 2025-06-19 13:39:00 +0700
categories: [bootloader-labs]
tags: [bootloader, x86, assembly, nasm, qemu, bios, boot process, low-level programming]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Mục lục
- [1. Mục lục](#1-mục-lục)
- [2. Quá trình khởi động máy tính](#2-quá-trình-khởi-động-máy-tính)
  - [2.1. Power Supply Unit (PSU)](#21-power-supply-unit-psu)
  - [2.2. Power On Self Test (POST)](#22-power-on-self-test-post)
  - [2.3. BIOS và Interrupt Vector Table](#23-bios-và-interrupt-vector-table)
  - [2.4. Boot Sector](#24-boot-sector)
- [3. Chuẩn bị môi trường](#3-chuẩn-bị-môi-trường)
- [4. Viết Bootloader đầu tiên](#4-viết-bootloader-đầu-tiên)
  - [4.1. Source code](#41-source-code)
  - [4.2. Giải thích từng dòng code](#42-giải-thích-từng-dòng-code)
- [5. Build và test Bootloader](#5-build-và-test-bootloader)
  - [5.1. Assemble code](#51-assemble-code)
  - [5.2. Tạo floppy image](#52-tạo-floppy-image)
  - [5.3. Chạy trên QEMU](#53-chạy-trên-qemu)

Bài viết này sẽ hướng dẫn bạn hiểu quá trình khởi động máy tính và viết bootloader đầu tiên cho kiến trúc x86.

## 2. Quá trình khởi động máy tính

### 2.1. Power Supply Unit (PSU)

Máy tính của chúng ta khởi động như thế nào? Chắc hẳn nhiều bạn từng tự hỏi điều gì thực sự xảy ra khi ta nhấn nút nguồn.

Có một thành phần gọi là **PSU (Power Supply Unit)** trong phần cứng của hệ thống. Khi bạn nhấn nút nguồn:

1. PSU nhận tín hiệu từ nút nguồn
2. Chuyển đổi dòng điện xoay chiều (AC) thành dòng điện một chiều (DC)
3. Cung cấp năng lượng cho các bộ phận khác của hệ thống
4. Gửi tín hiệu **'power-good'** đến BIOS khi điện áp ổn định

### 2.2. Power On Self Test (POST)

Khi nhận được tín hiệu 'power-good', BIOS bắt đầu quá trình **POST (Power On Self Test)**:

- Kiểm tra nguồn điện có đúng hay không
- Kiểm tra bộ nhớ có bị hỏng không  
- Phát hiện các thiết bị đang được kết nối
- Báo lỗi bằng mã số trên I/O port hoặc chuỗi tiếng bíp nếu có vấn đề

### 2.3. BIOS và Interrupt Vector Table

Sau khi POST hoàn tất, quyền điều khiển được chuyển đến **BIOS**:

1. BIOS tạo ra **Interrupt Vector Table (IVT)**
2. IVT lưu trữ địa chỉ của các Interrupt Service Routines
3. Interrupt là cơ chế phần cứng sử dụng để báo hiệu sự kiện cho CPU

### 2.4. Boot Sector

Công việc chính của BIOS là tải **bootloader**. Nhưng làm sao biết nó nằm ở đâu?

**Boot sector** là khu vực đặc biệt trên thiết bị lưu trữ:
- Thường là sector đầu tiên của disk (sector 0, track 0, head 0)
- BIOS load dữ liệu từ boot sector vào bộ nhớ tại địa chỉ **0x7c00**
- Sử dụng ngắt **0x19** để nhảy đến địa chỉ này

## 3. Chuẩn bị môi trường

Chúng ta cần cài đặt các công cụ cần thiết:

```bash
sudo apt-get install nasm qemu-system-x86
```

- **NASM**: Assembler cho Intel x86 architecture
- **QEMU**: Trình giả lập để test bootloader

## 4. Viết Bootloader đầu tiên

### 4.1. Source code

Tạo file `myboot.asm`:

```asm:myboot.asm
    org 0x7c00
    bits 16

Start: 
    cli
    hlt 

    times 510 - ($-$$) db 0
    dw 0xAA55
```

### 4.2. Giải thích từng dòng code

**`org 0x7c00`**
- Bootloader được load vào bộ nhớ tại địa chỉ 0x7c00
- Yêu cầu NASM set tất cả addresses theo địa chỉ này

**`bits 16`**
- Kiến trúc x86 khởi động ở **16-bit real mode**
- Có thể chuyển sang 32-bit protected mode sau này

**`cli` và `hlt`**
- `cli`: Clear all interrupts
- `hlt`: Halt the system
- Đảm bảo CPU không chạy random instructions

**`times 510 - ($-$$) db 0`**
- Một sector chỉ chứa **512 bytes**
- `$`: địa chỉ dòng hiện tại
- `$$`: địa chỉ dòng đầu tiên  
- `($-$$)`: kích thước chương trình
- Padding các byte còn lại bằng 0 để đủ 510 bytes

**`dw 0xAA55`**
- Hai byte cuối (511-512) lưu **boot signature**
- `0xAA` tại byte 511, `0x55` tại byte 512
- Cho BIOS biết sector này có thể boot được

## 5. Build và test Bootloader

### 5.1. Assemble code

Chuyển đổi assembly code sang machine code:

```bash
nasm -f bin -o myboot.bin myboot.asm
```

### 5.2. Tạo floppy image

Sao chép bootloader vào sector đầu tiên của floppy image:

```bash
dd status=noxfer conv=notrunc if=myboot.bin of=myboot.flp
```

### 5.3. Chạy trên QEMU

Khởi động bootloader đầu tiên của bạn:

```bash
qemu-system-i386 -fda myboot.flp
```

![QEMU Boot Screen](/assets/img/post/bootloader-x86/50b4d48c-8dff-4638-8545-0ac43b1f5179.png)
_Bootloader chạy thành công trên QEMU_

**Chúc mừng!** Bạn đã viết thành công bootloader đầu tiên cho kiến trúc x86. Mặc dù nó chỉ halt hệ thống, nhưng đây là bước đầu quan trọng để hiểu về low-level system programming.