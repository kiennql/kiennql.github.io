---
title: "USB: Nhập môn - Từ cơ bản đến nâng cao"
author: kiennql
date: 2025-06-23 16:36:00 +0700
categories: [usb, kernel, hardware]
tags: [linux, kernel, usb, driver, protocol, hardware, embedded]
description: "Hướng dẫn toàn diện về USB từ khái niệm cơ bản đến chi tiết giao thức."
math: true
mermaid: true
render_with_liquid: false
---


## 1. Nguồn gốc và phát triển của USB

### 1.1. USB là gì?

USB là viết tắt của **Universal Serial Bus** - một chuẩn giao tiếp "tuyến nối tiếp thông dụng".

So với USB, còn có nhiều loại bus khác như parallel bus, và các tuyến nối tiếp chuyên dụng khác. Để hiểu rõ vị trí của USB, ta có thể xem bảng so sánh sau:

![So sánh USB với các giao diện khác](/assets/img/post/usb-overview/image.png)
_**Hình 1.1: So sánh sự giống và khác nhau giữa USB và các loại bus khác**_

Từ bảng trên, USB không có ưu điểm nổi trội tuyệt đối, nhưng cân bằng tốt ở nhiều thông số. Điều quan trọng là ngoài USB, còn tồn tại rất nhiều giao diện khác với hình dáng phần cứng và lĩnh vực ứng dụng riêng biệt, không tương thích với nhau.

### 1.2. Tại sao cần có USB?

**Vấn đề trước USB:**
Trước khi USB xuất hiện, máy tính có quá nhiều loại giao diện khác nhau, gây khó khăn cho người dùng phổ thông:

![Các giao diện đa dạng phía sau thùng máy PC](/assets/img/post/usb-overview/image%201.png)
_**Hình 1.2: Các giao diện đa dạng phía sau thùng máy PC**_

- Đa dạng, phức tạp, khó sử dụng
- Không tương thích giữa các giao diện
- Cần cấu hình thủ công nhiều tham số

**Giải pháp USB:**
Khi USB ra đời, mọi thứ trở nên gọn gàng hơn:

![Các cổng kết nối sau khi có USB](/assets/img/post/usb-overview/image%202.png)
_**Hình 1.3: Các cổng kết nối phía sau thùng máy tính PC sau khi có giao diện USB**_

**Mục đích chính của USB:**

1. **Kết nối máy tính với điện thoại**: Tích hợp thiết bị máy tính và viễn thông
2. **Tiện lợi cho người dùng**: Hỗ trợ plug-and-play, không cần cấu hình phức tạp
3. **Mở rộng khả năng kết nối**: Một cổng cho nhiều ứng dụng khác nhau

**Tóm lại:** USB là giao diện "all-in-one", thay thế các cổng kết nối đa dạng trước đây bằng một chuẩn duy nhất, dễ sử dụng và hỗ trợ nhiều ứng dụng.

---

## 2. Kiến thức cơ bản về USB

Các thiết bị USB, từ cấu trúc logic vật lý, bao gồm hai phần: đầu **Host** và đầu **Device**.

Trong đó, đầu **Host** có bộ điều khiển phần cứng USB **Host Controller** tương ứng, còn đầu **Device** kết nối với thiết bị USB tương ứng.

### 2.1. Phần cứng liên quan đến USB

#### 2.1.1. Các loại bộ điều khiển USB: OHCI, UHCI, EHCI, xHCI

Do ảnh hưởng của lý do lịch sử, các bộ điều khiển USB đã xuất hiện nhiều loại khác nhau. Dù là loại nào, chúng đều phải tuân thủ các tiêu chuẩn tương ứng của USB.

**OHCI và UHCI (USB 1.1):**
- **OHCI** (Open Host Controller Interface): Được sáng lập bởi Compaq, Microsoft và National Semiconductor
- **UHCI** (Universal Host Controller Interface): Được sáng lập bởi Intel

**Sự khác biệt chính:**
- **OHCI**: Sử dụng phần cứng để thực hiện nhiều công việc hơn → phát triển driver dễ dàng hơn → thường dùng trong hệ thống nhúng
- **UHCI**: Để phần lớn chức năng cho phần mềm xử lý → phần cứng rẻ hơn → chủ yếu dùng trong PC

**Tại sao Intel thiết kế UHCI như vậy?**
Intel muốn tạo ra bộ điều khiển USB giá rẻ cho bo mạch chủ PC. Khi số lượng bo mạch chủ PC bán ra tăng, số lượng CPU Intel cũng tăng theo.

**Tại sao hệ thống nhúng ưa chuộng OHCI?**
Hệ thống nhúng có tài nguyên hạn chế, cần sử dụng tối thiểu tài nguyên CPU. OHCI giúp việc phát triển driver đơn giản hơn và tiết kiệm tài nguyên.

**Sự khác biệt kỹ thuật:**
1. **Số stage trong một frame**: OHCI có thể lên lịch nhiều stage, UHCI chỉ một stage
2. **Số transaction trong một frame**: OHCI có thể có nhiều transaction, UHCI tối đa một
3. **Tần suất quét**: OHCI quét ít nhất mỗi 32ms, UHCI có thể thấp hơn

**EHCI (USB 2.0):**
- Định nghĩa chuẩn bộ điều khiển chủ USB 2.0
- Xác định phần cứng cần thiết và chức năng phải thực hiện
- Định nghĩa ở cấp độ thanh ghi cho lập trình viên driver

**xHCI (USB 3.0):**
- Tương tự EHCI nhưng cho USB 3.0
- Định nghĩa cách thực hiện bộ điều khiển chủ USB 3.0

**Bảng 2.1. So sánh các loại bộ điều khiển USB**

| **Loại** | **Chuẩn USB** | **Tốc độ hỗ trợ** |
|----------|---------------|-------------------|
| **OHCI** | USB 1.1 | Low Speed và Full Speed |
| **UHCI** | USB 1.1 | Low Speed và Full Speed |
| **EHCI** | USB 2.0 | High Speed |
| **xHCI** | USB 3.0 | Super Speed |

#### 2.1.2. Định nghĩa chân của cổng USB

**Bảng 2.2. Định nghĩa chân cắm USB 1.x/2.0**

| **Chân** | **Tên** | **Màu dây** | **Mô tả** |
|----------|---------|-------------|-----------|
| 1 | VBUS | Đỏ | +5 V, nguồn cấp |
| 2 | D− | Trắng | Dữ liệu −, dây dữ liệu |
| 3 | D+ | Xanh lá cây | Dữ liệu +, dây dữ liệu |
| 4 | GND | Đen | Mặt đất, tiếp đất |

**Bảng 2.3. Định nghĩa chân cắm USB 3.0**

| **Chân** | **Màu** | **Tên tín hiệu ('A' connector)** | **Tên tín hiệu ('B' connector)** |
|----------|---------|----------------------------------|----------------------------------|
| 1 | Đỏ | VBUS | VBUS |
| 2 | Trắng | D− | D− |
| 3 | Xanh lá cây | D+ | D+ |
| 4 | Đen | GND | GND |
| 5 | Xanh dương | StdA_SSRX− | StdA_SSTX− |
| 6 | Vàng | StdA_SSRX+ | StdA_SSTX+ |
| 7 | Bọc chắn | GND_DRAIN | GND_DRAIN |
| 8 | Tím | StdA_SSTX− | StdA_SSRX− |
| 9 | Cam | StdA_SSTX+ | StdA_SSRX+ |
| Shell | Vỏ | Shield | Shield |

#### 2.1.3. Các loại đầu nối USB (connector)

Vì USB hỗ trợ nhiều ứng dụng khác nhau với yêu cầu kích thước giao diện khác nhau, nhiều loại giao diện USB đã được thiết kế.

**Thuật ngữ cơ bản:**
- **Đầu cắm** (plug): Cổng đực, cắm vào cái khác
- **Ổ cắm** (receptacle): Cổng cái, bị cắm vào

**Phân loại chính:**
Cổng USB được phân thành ba nhóm lớn:
1. **Type** - Loại thông thường
2. **Mini** - Phiên bản nhỏ gọn  
3. **Micro** - Phiên bản nhỏ hơn nữa

Mỗi nhóm chia thành hai loại: **Type A** và **Type B**

![Các loại connector USB](/assets/img/post/usb-overview/image%203.png)

**Bảng 2.4. Phân loại giao diện USB**

| **Giao diện** | **Phân loại** | **Đặc điểm** | **Mục đích sử dụng** |
|---------------|---------------|--------------|---------------------|
| **Type A** | Type | Hình chữ nhật | Máy tính cá nhân |
| **Type B** | Type | Hình vuông góc bo tròn | Thiết bị USB |
| **Mini A** | Mini | Thu nhỏ hình thang | Máy ảnh, ổ cứng di động |
| **Mini B** | Mini | Thu nhỏ hình chữ nhật | Thiết bị di động nhỏ |
| **Micro A** | Micro | Mỏng hơn Mini | Điện thoại di động |
| **Micro B** | Micro | Thu nhỏ hình chữ nhật | Điện thoại di động |

### 2.2. Phần mềm liên quan đến USB

Để thiết bị USB hoạt động bình thường, ngoài phần cứng còn cần hỗ trợ phần mềm tương ứng.

#### 2.2.1. Firmware của thiết bị USB

Đối với thiết bị USB, bên trong cần có firmware thực hiện các tác vụ cần thiết, chủ yếu là phản hồi các yêu cầu chuẩn và hoàn thành các thao tác đọc/ghi dữ liệu.

#### 2.2.2. Driver và phần mềm USB ở phía Host

Phía máy chủ (Host) cũng cần có driver tương ứng. Đối với Linux hay Windows, các driver thông dụng đã được triển khai sẵn, vì vậy thông thường không cần viết lại.

#### 2.2.3. Các phần mềm khác

Bao gồm các công cụ kiểm tra USB và phân tích giao thức để hỗ trợ phát triển và debug.

---

## 3. Tổng quan về giao thức USB

### 3.1. Tổng quan về nội dung giao thức USB 2.0

Giao thức USB hiện tại đã phát triển lên tới USB 3.0. Tuy nhiên, các thiết bị và công nghệ USB phổ biến hiện nay vẫn chủ yếu là USB 2.0. Vì vậy, tài liệu này chủ yếu sẽ giải thích các kiến thức cơ bản về giao thức USB dựa trên USB 2.0.

Về các tiêu chuẩn giao thức USB 2.0 và USB 3.0, bạn có thể tải xuống từ trang web chính thức: [https://www.usb.org/developers/docs/](https://www.usb.org/developers/docs/)

Chỉ riêng tài liệu về tiêu chuẩn USB 2.0 đã có tới 650 trang, và USB 3.0 với 482 trang. Tuy nhiên, phần nội dung liên quan trực tiếp đến giao thức USB chỉ khoảng 97 trang, là những gì chúng ta cần quan tâm.

### 3.2. Các phiên bản giao thức USB và tốc độ hỗ trợ

Giao thức USB đã trải qua 4 phiên bản chính thức: USB 1.1, USB 2.0, USB Wireless (2.5), và USB 3.0.

**Bảng 3.2. Sự phát triển các phiên bản giao thức USB**

| USB | Ngày phát hành | Thiết bị hỗ trợ | Tên gọi | Tốc độ tương ứng |
| --- | --- | --- | --- | --- |
| 1.1 | Tháng 8, 1998 | Có dây | Low Speed, Full Speed | 1.5Mbits/s, 12Mbits/s |
| 2.0 | Tháng 4, 2000 | Có dây | High Speed | 480Mbits/s |
| 2.5 | Tháng 9, 2010 | Không dây | Wireless USB 1.1 | - |
| 3.0 | Tháng 11, 2008 | Có dây | Super Speed | 5.0Gbits/s |

#### 3.2.1. Tại sao tốc độ USB ban đầu không được thiết kế nhanh hơn?

**USB 1.1** ban đầu chỉ hỗ trợ tốc độ thấp 1.5Mbits/s, mặc dù chậm nhưng đã đủ cho các thiết bị như chuột và bàn phím. Ưu điểm của tốc độ này là khả năng chống nhiễu điện từ (EMI) tốt hơn, giúp giảm chi phí thiết kế phần cứng.

**USB 2.0** ra đời để đáp ứng nhu cầu ngày càng cao về tốc độ truyền tải dữ liệu, chẳng hạn như việc sao chép nhạc từ MP3. Với USB 2.0, tốc độ có thể đạt khoảng 3-20MB/s, giúp giảm đáng kể thời gian sao chép dữ liệu.

**USB 3.0** ra đời để đáp ứng nhu cầu tương lai. Với USB 2.0, sao chép một bộ phim Blu-ray có thể mất vài phút, trong khi USB 3.0 có thể giảm thời gian này xuống chỉ còn vài giây hoặc vài chục giây.

### 3.3. Host là trung tâm của hệ thống USB

USB là giao thức mà thiết bị **Host** điều khiển toàn bộ quá trình truyền tải dữ liệu trên bus USB. Trên một bus USB duy nhất, chỉ có thể có một **Host**. USB không hỗ trợ nhiều Host cùng lúc.

Tuy nhiên, có một khái niệm đặc biệt là **USB OTG (On-The-Go)** để hỗ trợ nhiều thiết bị giao tiếp. USB OTG sử dụng **HNP (Host Negotiation Protocol)**, cho phép hai thiết bị thỏa thuận xem ai sẽ là Host. Ngay cả trong trường hợp OTG, vẫn chỉ có một Host tại mỗi thời điểm.

**USB Host** chịu trách nhiệm:
- Điều khiển tất cả các hoạt động truyền dữ liệu cơ sở
- Phân phối băng thông cho các thiết bị kết nối
- Sử dụng giao thức dựa trên vòng lệnh (token ring)

**Đặc điểm kỹ thuật:**
- **Topology**: Hình sao (star) - các thiết bị đều được kết nối vào một Hub
- **Số thiết bị tối đa**: 127 thiết bị trên cùng một bus USB
- **Số cổng thông thường**: 2-5 cổng cho mỗi USB Host
- **Host Controllers**: Có thể tích hợp nhiều controllers để cải thiện băng thông

### 3.4. Dữ liệu trong USB được mã hóa bằng NRZI

Dữ liệu được truyền qua USB sử dụng phương pháp mã hóa **NRZI (Non-Return-to-Zero Inverted)**.

#### 3.4.1. Mã hóa NRZI trong USB

Trong các giao thức truyền dữ liệu như UART, I2C, SPI, dữ liệu được truyền theo dạng nối tiếp. Vấn đề nảy sinh là sự đồng bộ giữa người gửi và người nhận, vì tốc độ hoạt động có thể không giống nhau.

Một giải pháp là đồng bộ hóa với tín hiệu đồng hồ. Ví dụ, trong **I2C**, dây **SDA** truyền dữ liệu, dây **SCL** truyền tín hiệu đồng hồ đồng bộ.

![image.png](/assets/img/post/usb-overview/image%204.png)
_**Hình 3.1. Định dạng mã hóa dữ liệu I2C**_

#### 3.4.2. Cách hoạt động của NRZI

NRZI là phương thức mã hóa dữ liệu phổ biến trong các giao thức nối tiếp, bao gồm USB:
- **"1"** được biểu thị bằng việc đảo chiều tín hiệu (từ cao sang thấp hoặc ngược lại)
- **"0"** được biểu thị bằng việc giữ nguyên tín hiệu

**So sánh các phương pháp mã hóa:**

**RZ (Return-to-Zero)**:
- Ba mức điện áp: dương (logic 1), âm (logic 0), và mức 0
- Tự đồng bộ nhưng lãng phí băng thông

![image.png](/assets/img/post/usb-overview/image%205.png)
_**Hình 3.2. Mã hóa về không**_

**NRZ (Non-Return-to-Zero)**:
- Hai mức điện áp: dương (logic 1) và âm (logic 0)
- Tiết kiệm băng thông nhưng khó đồng bộ hóa

![image.png](/assets/img/post/usb-overview/image%206.png)
_**Hình 3.3. Mã hóa không trở về 0 (NRZ)**_

**NRZI (Non-Return-to-Zero Inverted)**:
- Sử dụng sự đảo ngược tín hiệu để đại diện cho logic
- Trong USB: đảo ngược mức điện áp = logic 0, không thay đổi = logic 1
- Đặc biệt tiện lợi cho USB vì tín hiệu được truyền qua dây dẫn vi sai

![image.png](/assets/img/post/usb-overview/image%207.png)
_**Hình 3.4. NRZ và NRZI**_

#### 3.4.3. USB sử dụng Bit-Stuffing để đồng bộ tín hiệu xung nhịp

**Vấn đề đồng bộ hóa:**
- NRZ và NRZI đều không có đặc tính tự đồng bộ hóa
- USB giải quyết bằng trường đồng bộ (SYNC): 0000 0001
- Khi mã hóa NRZI, SYNC trở thành chuỗi dạng sóng vuông

**Giải pháp Bit-Stuffing:**
USB sử dụng **bit-stuffing** để giải quyết vấn đề đồng bộ:
- Nếu dữ liệu có 6 bit 1 liên tiếp, hệ thống sẽ chèn thêm một bit 0 bắt buộc
- Điều này khiến tín hiệu bắt buộc phải có sự đảo chiều
- Bộ thu có thể thực hiện điều chỉnh tần số một cách bắt buộc
- Bộ thu chỉ cần xóa các bit 0 được chèn sau chuỗi 6 bit 1 liên tiếp để khôi phục dữ liệu gốc

---

## 4. Chi tiết giao thức USB

### 4.1. USB Class

Về **Class** trong USB, đối với những người học giao thức USB, có lẽ đã từng nghe qua thuật ngữ này từ lâu.

Dưới đây là bảng phân loại cơ bản nhất về các **Class** của USB:

**Bảng 4.1. Bảng các Class của USB**

![image.png](/assets/img/post/usb-overview/image%208.png)

#### 4.1.1. Tại sao cần có nhiều loại USB Class?

USB được tạo ra để thay thế các cổng giao tiếp đa dạng và phức tạp trước đây bằng một chuẩn giao tiếp duy nhất. Để thay thế các cổng giao tiếp cũ, USB cần phải hỗ trợ hoặc thực hiện được các chức năng mà các giao tiếp cũ đã đảm nhiệm.

Vì vậy, khi thiết kế giao thức USB, các nhà phát triển đã đưa vào các định nghĩa tiêu chuẩn cho nhiều chức năng như chuột, bàn phím, lưu trữ dung lượng lớn, hình ảnh, v.v.

Do đó, có rất nhiều **Class** của USB, tức các phân loại dựa trên chức năng cụ thể. Mỗi **Class** đại diện cho một loại thiết bị và chức năng tương ứng. Ví dụ:

- Chuột và bàn phím thường thuộc **Class 3 - HID (Human Interface Device)**.
- Ổ USB (U-disk) thuộc **Class 8 - Mass Storage**.

Quay trở lại bảng **4.1 'Bảng các Class của USB'**, cột 'Descriptor Usage' (Sử dụng Descriptor) liệt kê các giá trị như **Interface** hoặc **Device**, tương ứng với thông tin từ bảng **4.2 'Loại Descriptor của USB'**.

**Bảng 4.2. Các loại Descriptor của USB**

![image.png](/assets/img/post/usb-overview/image%209.png)

**Về mối quan hệ giữa Device, Interface và các thành phần khác:**

- **Device ⇒ Configuration ⇒ Interface ⇒ Endpoint**

### 4.2. Khung (framework) của USB

Về cấu trúc khung của USB, dưới đây là một số hình ảnh liên quan:

![image.png](/assets/img/post/usb-overview/image%2010.png)
_**Hình 4.1. USB Implementation Areas**_

![image.png](/assets/img/post/usb-overview/image%2011.png)
_**Hình 4.2. USB Physical Bus Topology**_

![image.png](/assets/img/post/usb-overview/image%2012.png)
_**Hình 4.3. USB Logical Bus Topology**_

![image.png](/assets/img/post/usb-overview/image%2013.png)
_**Hình 4.4. USB Communication Flow**_

![image.png](/assets/img/post/usb-overview/image%2014.png)
_**Hình 4.5. USB Layers in Linux**_

### 4.3. USB Transfer và Transaction

![image.png](/assets/img/post/usb-overview/image%2015.png)
_**Hình 4.6. USB Transfer and Transaction**_

### 4.4. USB Enumeration (quá trình định danh USB)

Về quá trình USB Enumeration, đây là nội dung cơ bản và quan trọng nhất mà người học giao thức USB cần nắm vững.

#### 4.4.1. USB Enumeration là gì?

USB Enumeration, hay còn gọi là USB Emulation, nghĩa đen là liệt kê các thiết bị USB. Tuy nhiên, thực chất, đây là quá trình **khởi tạo thiết bị USB**.

Cụ thể:

- **Quá trình Enumeration** là cuộc trao đổi giữa USB Host và USB Device.
- USB Host nhận các tham số do USB Device gửi lên để xác định:
    - Loại thiết bị USB là gì.
    - Các chức năng mà thiết bị hỗ trợ.
- Dựa trên các thông tin đó, Host sẽ khởi tạo các tham số cần thiết để thiết bị có thể hoạt động bình thường.

Tóm lại, USB Enumeration chính là quá trình **khởi tạo và chuẩn bị** để một thiết bị USB bắt đầu hoạt động.

#### 4.4.2. Quá trình USB Enumeration diễn ra như thế nào?

**USB Enumeration Process on Windows (Quá trình USB Enumeration trên Windows)**

Dưới đây là tóm tắt về quá trình USB Enumeration trên Windows:

1. **Host hoặc Hub phát hiện kết nối của thiết bị mới** thông qua các điện trở pull-up trên cặp dữ liệu của thiết bị.
2. **Host chờ ít nhất 100ms**, cho phép thiết bị được cắm hoàn toàn và đảm bảo nguồn điện ổn định.
3. **Host phát lệnh reset**, đặt thiết bị vào trạng thái mặc định. Thiết bị có thể phản hồi địa chỉ mặc định là 0.
4. **Windows Host yêu cầu 64 byte đầu tiên của Device Descriptor**.
5. Sau khi nhận 8 byte đầu tiên của Device Descriptor, host sẽ **phát lệnh reset lại bus**.
6. **Host phát lệnh Set Address**, đặt thiết bị vào trạng thái có địa chỉ.
7. **Host yêu cầu toàn bộ 18 byte của Device Descriptor**.
8. Tiếp theo, **host yêu cầu 9 byte của Configuration Descriptor** để xác định kích thước tổng thể của cấu hình.
9. **Host yêu cầu 255 byte của Configuration Descriptor**.
10. Host yêu cầu **bất kỳ String Descriptors nào nếu có**.
11. **Cuối cùng, Windows sẽ yêu cầu driver cho thiết bị**. Sau đó, hệ thống sẽ yêu cầu tất cả các descriptors một lần nữa trước khi phát lệnh Set Configuration.

#### 4.4.3. Phân tích ví dụ chi tiết về quá trình Enumeration của USB

Dữ liệu liên quan đến quá trình **USB enumeration** trong phần này được lấy từ một dự án phát triển trước đó, qua việc sử dụng **SBAE USB**. Phần mềm này có khả năng ghi lại dữ liệu USB và phân tích chi tiết từng byte của dữ liệu.

**Dữ liệu ví dụ về USB Enumeration:**

Dữ liệu thu được từ công cụ **sniffer** trong quá trình USB enumeration có tổng cộng 66 byte (0x42 byte):

`0902420002010480E10904000002FF000000092110010001223F0007050103400001070581034000010904010001030000000921100100012221000705820340000A`

Dữ liệu này có thể được phân thành 8 nhóm như sau:

1. `0902420002010480E1` - **Configuration descriptor**
2. `0904000002FF000000` - **Interface descriptor**
3. `092110010001223F00` - **Class descriptor**
4. `07050103400001` - **Endpoint descriptor**
5. `07058103400001` - **Endpoint descriptor**
6. `090401000103000000` - **Interface descriptor**
7. `092110010001222100` - **HID descriptor**
8. `0705820340000A` - **Endpoint descriptor**

**Phân tích từng trường dữ liệu cụ thể:**

**1. Configuration Descriptor: 0902420002010480E1**

![image.png](/assets/img/post/usb-overview/image%2016.png)
_**Bảng 4.3. USB Configuration Descriptors**_

![image.png](/assets/img/post/usb-overview/image%2017.png)
_**Hình 4.7. Mô tả Cấu hình (Configuration Descriptor): 0902420002010480E1**_

**2. Interface Descriptor: 0904000002FF000000**

![image.png](/assets/img/post/usb-overview/image%2018.png)
_**Bảng 4.4. USB Interface Descriptors**_

![image.png](/assets/img/post/usb-overview/image%2019.png)
_**Hình 4.8. Mô tả Interface: 0904000002FF000000**_

**4,5. Endpoint Descriptors**

![image.png](/assets/img/post/usb-overview/image%2020.png)
_**Bảng 4.5. USB Endpoint Descriptors**_

![image.png](/assets/img/post/usb-overview/image%2021.png)
_**Hình 4.9. Endpoint (Interrupt Out) Descriptor: 07050103400001**_

![image.png](/assets/img/post/usb-overview/image%2022.png)
_**Hình 4.10. Endpoint (Interrupt In) Descriptor: 07058103400001**_

**6,7. Interface và HID Descriptors**

![image.png](/assets/img/post/usb-overview/image%2023.png)
_**Hình 4.11. Interface Descriptor: 090401000103000000**_

Do **bInterfaceClass = 0x03** tương ứng với **HID (Human Interface Device)**, nên việc phân tích nội dung phần này sẽ dựa trên định nghĩa trong giao thức HID.

![image.png](/assets/img/post/usb-overview/image%2024.png)
_**Bảng 4.6. USB HID Descriptors**_

![image.png](/assets/img/post/usb-overview/image%2025.png)
_**Bảng 4.7. USB HID Descriptor: 092110010001222100**_

**8. Endpoint Descriptor**

![image.png](/assets/img/post/usb-overview/image%2026.png)
_**Bảng 4.12. Endpoint (Interrupt In 2) Descriptor: 0705820340000A**_

### 4.5. Ghi chú về USB OHCI

1. **OHCI** chủ yếu được sử dụng trong các hệ thống nhúng, vì đặc điểm của nó là thực hiện tối đa các chức năng bằng phần cứng, giúp phần mềm điều khiển USB trở nên đơn giản hơn nhiều và thống nhất hơn.

2. **Endpoint Descriptor (ED)**: Là một cấu trúc dữ liệu được lưu trong bộ nhớ, mô tả cách bộ điều khiển chủ (HC) và thiết bị tương tác với nhau. Trong ED có chứa một con trỏ trỏ đến **TD** (*Transfer Descriptor*, mô tả truyền dữ liệu).

3. **Các từ viết tắt thường gặp:**
- **HC:** Host Controller (Bộ điều khiển chủ)
- **HCCA:** Host Controller Communication Area (Vùng giao tiếp của bộ điều khiển chủ)
- **HCD:** Host Controller Driver (Trình điều khiển bộ điều khiển chủ)
- **HCDI:** Host Controller Driver Interface (Giao diện trình điều khiển bộ điều khiển chủ)
- **HCI:** Host Controller Interface (Giao diện bộ điều khiển chủ)

4. **Mối quan hệ giữa phần mềm và phần cứng:**

![image.png](/assets/img/post/usb-overview/image%2027.png)
_**Hình 4.13. Mối quan hệ giữa phần mềm và phần cứng trong USB host**_

5. **SOF (Start Of Frame)**: Chức năng của SOF là giúp các điểm cuối (endpoints) nhận diện sự bắt đầu của một khung (frame), và đồng bộ hóa thời gian bắt đầu và kết thúc của các điểm cuối nội bộ, giúp chúng đồng bộ với Host.

6. Trong **OHCI**, dữ liệu truyền được chia thành hai loại: **chu kỳ (periodic)** và **không chu kỳ (non-periodic)**:
   - Các truyền chu kỳ bao gồm truyền ngắt (Interrupt Transfer) và truyền đồng bộ (Isochronous Transfer)
   - Truyền không chu kỳ bao gồm truyền điều khiển (Control Transfer) và truyền lớn (Bulk Transfer)

7. **Kênh giao tiếp giữa HC và HCD:**

![image.png](/assets/img/post/usb-overview/image%2028.png)
_**Hình 4.14. USB Communication Channel**_

8. **Cấu trúc danh sách liên kết:**

![image.png](/assets/img/post/usb-overview/image%2029.png)
_**Hình 4.15. USB Typical List Structure**_

Trong **HCCA** có một con trỏ đầu (head pointer), trỏ đến một danh sách liên kết **ED** (Endpoint Descriptor). Mỗi **ED** trong danh sách này chứa một danh sách liên kết **TD** (Transfer Descriptor), với một hoặc nhiều **TD** cần được xử lý.

---