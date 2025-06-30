---
title: "AARCH64 Assembly - Khởi đầu (Phần 1)"
author: kiennql
date: 2025-06-27 11:13:00 +0700
categories: [asm, aarch64]
tags: [aarch64, assembly, cpu, registers, risc]
description: "Nhập môn AARCH64 Assembly: registers, instructions, memory operations cơ bản."
math: true
mermaid: true
render_with_liquid: false
---

##  1. Khởi đầu

Trong phần này, chúng ta sẽ bắt đầu khám phá ngôn ngữ **assembly AARCH64**.

###  1.1. Registers

Vào buổi bình minh của thời đại, các **bộ xử lý trung tâm (CPU)** có thể hoạt động trực tiếp trên **bộ nhớ** vì cả hai đều có tốc độ tương đối giống nhau. **CPU** trở nên nhỏ hơn và nhanh hơn, khiến tốc độ của **bộ nhớ** bị bỏ lại phía sau, đồng thời **bộ nhớ** cũng ngày càng xa hơn.

Một **CPU** có thể nằm trên một **board** trong khi **RAM** nằm trên **board** khác và phải được truy cập qua một **bus** chung. **CPU** tiếp tục trở nên nhỏ hơn và nhanh hơn, có thể nằm trên một **chip** trong khi **RAM** nằm trên một tập hợp **chip** khác.

Ý tưởng về **registers** được giới thiệu từ rất lâu như là **super fast storage** được triển khai trực tiếp trong **CPU**. Vì chúng nằm trong **CPU**, nên khoảng cách không thực sự là vấn đề. Tương tự, vì chúng nằm trong **CPU**, chúng hoạt động với tốc độ của chính **CPU**.

**Registers** không có **địa chỉ** vì chúng không nằm trong **bộ nhớ**. Thay vào đó, chúng có **names** và **naming conventions**. Chúng chỉ có khái niệm tối thiểu về **kiểu dữ liệu** vì ngoài **integer**, **floating point** (single hoặc double precision) và **pointer**, mọi khái niệm "type" khác đều là **syntactic sugar** được cung cấp bởi ngôn ngữ và **compiler** của bạn.

Bộ xử lý **ARM** là bộ xử lý **RISC** (**Reduced Instruction Set Computers**). Ý tưởng cơ bản đằng sau **RISC** là làm cho các **instruction** đơn giản hơn để tạo chỗ cho nhiều **registers** hơn. **AARCH64** (cái mà chúng ta đang học) có rất nhiều **registers**; **32 integer registers** và **32 floating point registers**.

Ngoài ra, các **floating point registers** cũng có thể phục vụ như **vector** hoặc **SIMD registers** (sẽ học sau).

Một số **registers** được dành riêng cho các mục đích cụ thể (sẽ học sau).

`rn` có nghĩa là **register** "of some type" number n.

Loại **register** được chỉ định bằng một **chữ cái**. **Register** nào trong một loại nhất định được chỉ định bằng một **số**. Có một số ngoại lệ cho điều này.

Đây là tóm tắt giới thiệu:

| **Chữ cái** | **Loại** |
| ------- | ----- |
| **x** | **64 bit integer** hoặc **pointer** |
| **w** | **32 bit** *hoặc nhỏ hơn* **integer** |
| **d** | **64 bit floats** (**doubles**) |
| **s** | **32 bit floats** |

Một số loại **register** đã được bỏ qua.

####  1.1.1. Integers

| **Khai báo integer này** | **Đây LÀ một integer** |
| -------------------- | ------------------ |
| **char**  | **wn** |
| **short** | **wn** |
| **int**   | **wn** |
| **long**  | **xn** |

**Registers** không cần phải được khai báo. Chúng đơn giản **LÀ** có sẵn.

Có **32 integer registers** nhưng một số, như **x30** được sử dụng cho các mục đích cụ thể. **x31** cũng không khả dụng.

Khi bạn muốn một phép toán **integer 64 bit**, bạn sử dụng **x register**. Đối với tất cả các phép toán **integer** khác, bạn sử dụng **w register** và chỉ định thêm kích thước bằng cách sử dụng các **instruction** khác nhau, như bạn sẽ thấy sau.

Các **x** và **w registers** chiếm cùng một không gian. **w0** là nửa dưới của **x0** chẳng hạn. Ghi vào **x0** sẽ ghi đè **w0** và ngược lại.

####  1.1.2. Pointers

| **Khai báo pointer này** | **Đây LÀ một pointer** |
| -------------------- | ------------------ |
| ***type* *** | **xn** |

Tất cả **pointers** được lưu trữ trong **x registers**. **X registers** dài **64 bits** nhưng nhiều **hệ điều hành** không hỗ trợ **không gian địa chỉ 64 bit** vì việc theo dõi **không gian địa chỉ** lớn như vậy bản thân nó sẽ sử dụng rất nhiều không gian. Thay vào đó, các **OS** thường có **không gian địa chỉ 48 đến 52 bit**. Vì vậy, trong khi một **pointer** nằm trong một **register 64 bit**, có thể không phải tất cả các **bit** đều thực sự được sử dụng trong việc tạo **pointer**.

####  1.1.3. Floats

Có **32 registers** bổ sung cho các giá trị **floating point** từ **half floats** đến **single precision** đến **double precision** và hơn thế nữa. Cái gì vượt ra ngoài **double**? **Vector registers** có thể hỗ trợ nhiều giá trị trong một **register** "rất lớn" duy nhất.

Nếu bạn chưa bao giờ nghe về ***half floats***, đừng lo lắng. Sau khóa học này, bạn có thể sẽ không bao giờ nghe về chúng nữa.

###  1.2. Instructions

Bạn còn nhớ **RISC** nghĩa là gì không? Nó là viết tắt của **Reduced Instruction Set Computer** – tức là kiến trúc với tập lệnh đơn giản. Điều này đúng trong quá khứ, nhưng hiện nay kiến trúc **AArch64** của ARM đã có một bộ lệnh rất lớn – gồm **hàng trăm lệnh**, và mỗi lệnh có thể có **nhiều biến thể** khác nhau. Nghe có vẻ áp lực, nhưng đừng lo – bạn **không cần học hết**. Chỉ cần hiểu và nắm vững những lệnh quan trọng, bạn vẫn có thể viết được những chương trình phức tạp.

Một điều đặc biệt trong **AArch64** là mọi **lệnh (instruction)** đều có độ dài **chính xác 4 byte** (tức **32-bit**). CPU sẽ **giải mã lệnh** dựa hoàn toàn vào 4 byte đó – từ việc biết đây là lệnh gì, phiên bản nào của lệnh, sử dụng thanh ghi nào, và thao tác với dữ liệu gì. Tất cả đều được **mã hóa trong đúng 4 byte** này.

Bạn có thể sẽ tự hỏi:  
> “Nếu một lệnh luôn chỉ có 4 byte, vậy làm sao để load được một số nguyên rất lớn (ví dụ 64-bit) vào thanh ghi?”

Đó là câu hỏi hợp lý – và sẽ có lời giải thích ở phần sau. **Hãy kiên nhẫn.**

Khi so sánh với kiến trúc **x86/x64 của Intel**, sự khác biệt hiện rõ: Intel dùng kiểu **CISC (Complex Instruction Set Computer)**, cho phép độ dài lệnh **thay đổi**, từ **1 đến 15 byte**. Điều này mang lại tính linh hoạt cao, nhưng cũng làm cho việc giải mã lệnh trở nên **phức tạp hơn rất nhiều**.

Cuối cùng, mỗi lệnh được thể hiện bằng một **mnemonic** – tức là tên viết tắt gồm một vài chữ cái (ví dụ `MOV`, `ADD`, `STR`...), và **assembler** sẽ chuyển những mnemonic đó thành **mã máy (opcode)** để CPU thực thi.

Hầu hết (nhưng không phải tất cả) các **instruction AARCH64** có ba ***operands***. Chúng được đọc theo cách sau:

```asm
    op     ra, rb, rc
```

có nghĩa là:

```asm
    ra = rb op rc
```

Ví dụ cụ thể:

```asm
    sub    x0, x0, x1
```

có nghĩa là:

```asm
    x0 = x0 - x1
```

Một ví dụ về **instruction hai operand** là:

```asm
    mov    x0, x1
```

Điều này có nghĩa là ***copy*** nội dung **64 bit** của **x1** vào **register 64 bit x0**.

Hoặc:

```asm
    x0 = x1
```

###  1.3. Trộn lẫn các loại Register

Trong hầu hết các trường hợp, bạn **không thể trộn các loại thanh ghi (register) có kích thước khác nhau trong cùng một lệnh**.  
Ví dụ, bạn **không thể cộng một thanh ghi 64-bit (`x`) với một thanh ghi 32-bit (`w`)** trong cùng một câu lệnh – điều đó sẽ gây lỗi.

#### 1.3.1. Ví dụ

```asm
mov    w0, 10       // Gán giá trị 10 (32-bit) vào thanh ghi w0
```

Bạn **không thể làm như sau**:

```asm
add    x1, w0, x1   // Sai! Không thể trộn w0 (32-bit) và x1 (64-bit)
```

Thay vào đó, bạn cần dùng **các thanh ghi cùng loại**, ví dụ đều là `x` (64-bit):

```asm
add    x1, x0, x1   // Hợp lệ - tất cả đều là thanh ghi 64-bit
```

---

Khi bạn gán một giá trị **nhỏ hơn 64-bit** (ví dụ 32-bit) vào một thanh ghi kiểu `w`, thì **các bit cao hơn trong thanh ghi `x` tương ứng sẽ tự động bị đặt về 0**.

Điều này có nghĩa là:

- Nếu bạn ghi vào `w0`, thì `x0` cũng được cập nhật.
- Bit từ `0 đến 31` chứa giá trị mới.
- Bit từ `32 đến 63` sẽ được **đặt thành 0**.

> Đây là **zero-extension**, không phải **sign-extension**.

Vì vậy, bạn có thể **an toàn sử dụng `x0`** sau khi vừa gán giá trị vào `w0` – kết quả vẫn đúng như mong đợi.

###  1.4. Hai Instructions để xử lý Memory

Với rất ít ngoại lệ, **kiến trúc AArch64** chỉ cho phép thực hiện các phép toán trên **dữ liệu nằm trong các thanh ghi (register)**.  
Điều đó có nghĩa là bạn **không thể thao tác trực tiếp lên dữ liệu trong RAM bằng các phép toán cộng, trừ, nhân...** như trên một số kiến trúc khác.

Thay vào đó, nếu muốn làm việc với dữ liệu trong bộ nhớ, bạn cần sử dụng hai loại lệnh riêng biệt:

- **Lệnh nạp (load)**: chuyển dữ liệu **từ RAM vào thanh ghi**
- **Lệnh lưu (store)**: chuyển dữ liệu **từ thanh ghi vào RAM**

Cả hai loại lệnh này đều cần chỉ rõ:
- **Thanh ghi (register)** dùng để lưu/đọc dữ liệu
- **Địa chỉ trong RAM** mà dữ liệu được đọc từ hoặc ghi vào – địa chỉ này phải nằm trong một **thanh ghi kiểu `x` (64-bit)**

---

#### 1.4.1. Lệnh Load (`ldr`)

Cú pháp tổng quát:

```asm
ldr    rn, [xm]
```

Tương đương trong ngôn ngữ C:

```c
type *ptr = some_address;
type var;
var = *ptr;
```

Hiểu đơn giản: lấy giá trị tại địa chỉ `some_address` trong RAM, và gán nó vào `rn`.

---

#### 1.4.2. Lệnh Store (`str`)

Cú pháp tổng quát:

```asm
str    rn, [xm]
```

Tương đương trong C:

```c
type *ptr = some_address;
type var;
*ptr = var;
```

Hiểu đơn giản: ghi giá trị từ thanh ghi `rn` vào địa chỉ `some_address` trong RAM.

---

Tóm lại:  
Bạn **phải luôn nạp dữ liệu vào thanh ghi trước khi tính toán**, và **phải lưu lại vào RAM nếu muốn giữ kết quả**.  
Các phép toán **chỉ xảy ra trong register** – RAM chỉ là nơi lưu trữ trung gian.

---

#### 1.4.3. Ghi chú mở rộng

- Các phép trên chỉ là **tương đương logic**, không phải dịch đúng 100% – nhưng gần đúng để bạn dễ hình dung.
- Bạn cũng có thể **nạp hoặc lưu cùng lúc hai thanh ghi** bằng các lệnh:
  - `stp` – store pair (lưu 2 register cùng lúc)
  - `ldp` – load pair (nạp 2 register cùng lúc)
- Ngoài ra, AArch64 hỗ trợ:
  - **Post-increment / Post-decrement**: tự động tăng/giảm địa chỉ sau khi truy cập
  - **Pre-increment / Pre-decrement**: tự động tăng/giảm địa chỉ trước khi truy cập

Các tính năng này rất hữu ích trong thao tác với mảng, stack, hay khi duyệt qua nhiều ô nhớ liên tiếp.
