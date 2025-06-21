---
title: Tạo toolchain cho BeagleBone Black bằng crosstool-NG (Phần 1)
author: kiennql
date: 2025-06-10 0:33:00 +0700
categories: [bbb-labs]
tags: [beaglebone black, embedded linux, crosstool-ng, uclibc, toolchain, arm, cross-compile]
math: true
mermaid: true
render_with_liquid: false
---

## 1. Trình tạo Toolchain

Crosstool-NG là công cụ tạo **cross-compiling toolchain**.

Cross-compiler tương tự như gcc/g++, nhưng tạo ra file thực thi cho kiến trúc khác — ở đây là ARM.

Toolchain cần một số thành phần đặc biệt:

* **Linux kernel headers** để hỗ trợ giao tiếp với kernel.
* Thư viện C phù hợp, trong dự án này là **uClibc** — một thư viện nhỏ gọn, tối ưu cho các hệ thống nhúng.

## 2. Biên dịch crosstool-NG

Đầu tiên, clone git repository.

```bash
git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng/
```

Tiếp theo, chúng ta cần cấu hình và build source code. Có hai cách để làm điều này, tuy nhiên, tôi sẽ làm theo cách "hacker" cho phép chúng ta chạy cục bộ, thay vì export vào PATH.

```bash
./bootstrap
./configure --enable-local
```

Có thể bạn sẽ cần chạy script configure này nhiều lần, cài đặt các gói còn thiếu trong quá trình. Nếu bạn không sử dụng Ubuntu như tôi, bạn có thể cần Google hoặc ChatGPT một chút để tìm ra tên gói thay thế.

Sau khi xử lý xong, bạn có thể biên dịch.

```bash
make
```

## 3. Tạo toolchain

Bạn có thể xem menu trợ giúp bằng cách nhập:

```bash
./ct-ng help
```

Chúng ta sử dụng cortex a8 vì điều này sẽ loại bỏ phần lớn cấu hình mà chúng ta có thể cần làm. Bạn có thể xem tất cả các mẫu với:

```bash
./ct-ng list-samples
```

Chúng ta chọn cortex a8, vậy:

```bash
./ct-ng arm-cortex_a8-linux-gnueabi
```

Được rồi, bây giờ hãy vào menu và cấu hình toolchain:

```bash
./ct-ng menuconfig
```

Menuconfig sẽ trông như thế này:

![](/assets/img/post/toolchain-bbb/605516D5-ADDC-47FF-BEA1-65F18920137C.png)

Cấu hình toolchain như sau:

**Trong Path and misc options:**
- Thay đổi Maximum log level to see thành **DEBUG** (tìm **LOG_DEBUG** trong giao diện, sử dụng phím **/**) để chúng ta có thể có thêm chi tiết về những gì đã xảy ra trong quá trình build trong trường hợp có sự cố.

**Trong Target options:**
- Đặt Use specific FPU (**ARCH_FPU**) thành **vfpv3**.
- Đặt Floating point thành hardware (**FPU**).

**Trong Toolchain options:**
- Đặt Tuple's vendor string (**TARGET_VENDOR**) thành **training**.
- Đặt Tuple's alias (**TARGET_ALIAS**) thành **arm-linux**. Bằng cách này, chúng ta sẽ có thể sử dụng compiler như arm-linux-gcc thay vì arm-training-linux-uclibcgnueabihf-gcc, dài hơn nhiều để gõ.

**Trong Operating System:**
- Đặt Version of linux thành phiên bản **5.15.x** được đề xuất. Chúng ta chọn phiên bản này vì nó khớp với phiên bản kernel mà chúng ta sẽ chạy trên board. Ít nhất, phiên bản của kernel headers không mới hơn.

**Trong C-library:**
- Nếu chưa được đặt, đặt C library thành **uClibc-ng** (**LIBC_UCLIBC_NG**)
- Giữ phiên bản mặc định được đề xuất
- Nếu cần, bật Add support for IPv6 (**LIBC_UCLIBC_IPV6**), Add support for WCHAR (**LIBC_UCLIBC_WCHAR**) và Support stack smashing protection (**SSP**) (**LIBC_UCLIBC_HAS_SSP**)

**Trong C compiler:**
- Đặt Version of gcc thành **11.3.0**. Chúng ta cần sử dụng gcc 11.x, vì Buildroot 2022.02 mà chúng ta sẽ dung sau này chưa hỗ trợ gcc 12.x toolchains: gcc 12.x được phát hành sau Buildroot 2022.02.
- Đảm bảo rằng C++ (**CC_LANG_CXX**) được bật

**Trong Debug facilities:**
- Tắt **duma**, **gdb**, **ltrace**. Chúng sẽ không cần thiết cho các bài lab sẽ sử dụng toolchain này.
- Chỉ giữ strace (**DEBUG_STRACE**) được bật

Bây giờ, chúng ta đã sẵn sàng để build toolchain:

```bash
./ct-ng build
```

Điều này có thể mất một lúc, vì chúng ta đang build toàn bộ arm compiler, linker, debugger, v.v.

Bây giờ nó đã được build, bạn nên thêm `$HOME/x-tools/arm-training-linux-uclibcgnueabihf/bin` vào PATH của bạn. Cách tôi ưa thích để làm điều này là sửa đổi file `~/.bashrc` để định nghĩa PATH.

Log khi build thành công sẽ như thế này:

![](/assets/img/post/toolchain-bbb/3B20CE99-590F-49DD-9AD4-2E80FEE12852.png)

## 4. Xác nhận toolchain hoạt động

Viết file C đơn giản như này:

```c
#include <stdlib.h>
#include <stdio.h>

int main (void)
{
    printf( "Hello world!\n" );
    return EXIT_SUCCESS;
}
```

Tạo file `hello.c` và copy nội dung này vào.

Thử biên dịch với:

```bash
arm-linux-gcc -o hello hello.c
```

Nếu bạn nhận được `bash: arm-linux-gcc: command not found...` có nghĩa là bạn chưa thêm compiler vào PATH đúng cách.

Nếu không, bây giờ bạn sẽ thấy file C đã biên dịch. Thông thường, bạn sẽ chạy file này với `./hello`, nhưng bạn sẽ nhận được `bash: ./hello: cannot execute binary file: Exec format error`. Điều này là do chúng ta đã biên dịch binary này thành ARM thay vì X86, đó chính là điều chúng ta muốn!! :)

Để kiểm tra điều này, chúng ta sẽ cần một ARM VM. qemu là lựa chọn rõ ràng ở đây. Hãy cài đặt nó với:

```bash
# Cho người dùng Ubuntu/Debian  
sudo apt install qemu-user
```

Thử chạy binary với:

```bash
qemu-arm hello
```

Bạn sẽ nhận được lỗi `/lib/ld-uClibc.so.0: No such file or directory`. Điều này là do qemu thiếu thư viện C của chúng ta, uClibc. Điều này đã được biên dịch thành object file bởi cấu hình của chúng ta với crosstool-NG. Nói với qemu nơi tìm nó:

```bash
qemu-arm -L ~/x-tools/arm-training-linux-uclibcgnueabihf/arm-training-linux-uclibcgnueabihf/sysroot hello
```

Bây giờ bạn sẽ thấy dòng kinh điển:

```
Hello world!
```

Chúc mừng! Bạn đã biên dịch chương trình ARM đầu tiên với toolchain mà bạn đã tự tạo!
