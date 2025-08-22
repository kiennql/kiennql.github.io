---
title: "LFS: Bộ lập lịch - Scheduler (Phần 4)"
author: kiennql
date: 2025-08-22 10:30:00 +0700
categories: [pi-labs]
tags: [scheduler, raspberry-pi, linux kernel, arm, lfs]
description: "Hướng dẫn triển khai bộ lập lịch (scheduler) trong RPi OS - từ task_struct, thuật toán scheduling đến context switch và quản lý bộ nhớ."
math: true
mermaid: true
render_with_liquid: false
---

## 4. Bộ lập lịch (Scheduler)

Đến thời điểm này, PRi OS đã là một chương trình **bare metal** khá phức tạp, nhưng thành thật mà nói, chúng ta vẫn chưa thể gọi nó là một hệ điều hành. Lý do là vì nó chưa thể thực hiện bất kỳ nhiệm vụ cốt lõi nào mà một OS nào cũng phải làm được. Một trong những nhiệm vụ cốt lõi đó được gọi là **process scheduling** (lập lịch tiến trình). Bằng **scheduling** tôi có ý là một hệ điều hành phải có khả năng chia sẻ thời gian CPU giữa các **process** khác nhau. Phần khó khăn là một **process** phải không nhận biết được việc **scheduling** đang diễn ra: nó phải coi mình là thứ duy nhất đang chiếm dụng CPU. Trong bài học này, chúng ta sẽ thêm chức năng này vào RPi OS.

### 4.1. task_struct

Nếu chúng ta muốn quản lý các **process**, điều đầu tiên cần làm là tạo một **struct** mô tả một **process**. Linux có một **struct** như vậy và nó được gọi là `task_struct` (trong Linux cả **thread** và **process** đều chỉ là các loại **task** khác nhau). Vì chúng ta chủ yếu bắt chước cách triển khai của Linux, chúng ta sẽ làm tương tự. RPi OS [task_struct](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/sched.h#L36) trông như sau:

```c
struct cpu_context {
    unsigned long x19;
    unsigned long x20;
    unsigned long x21;
    unsigned long x22;
    unsigned long x23;
    unsigned long x24;
    unsigned long x25;
    unsigned long x26;
    unsigned long x27;
    unsigned long x28;
    unsigned long fp;
    unsigned long sp;
    unsigned long pc;
};

struct task_struct {
    struct cpu_context cpu_context;
    long state;
    long counter;
    long priority;
    long preempt_count;
};
```

**Struct** này có các thành viên sau:

* `cpu_context` Đây là một **structure** nhúng chứa các giá trị của tất cả **register** có thể khác nhau giữa các **task** đang được chuyển đổi. Một câu hỏi hợp lý cần đặt ra là tại sao chúng ta chỉ lưu một số **register** `x19 - x30` và `sp` chứ không phải tất cả? (`fp` là `x29` và `pc` là `x30`) Câu trả lời là việc **context switch** thực tế chỉ xảy ra khi một **task** gọi hàm [cpu_switch_to](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.S#L4). Vậy nên, từ góc độ của **task** đang được chuyển đổi, nó chỉ gọi hàm `cpu_switch_to` và nó trả về sau một khoảng thời gian (có thể dài). **Task** không nhận ra rằng một **task** khác đã chạy trong khoảng thời gian này. Theo **ARM calling conventions**, các **register** `x0 - x18` có thể bị ghi đè bởi hàm được gọi, vì vậy người gọi không được giả định rằng giá trị của những **register** này sẽ tồn tại sau một lần gọi hàm. Đó là lý do tại sao không có ý nghĩa gì khi lưu các **register** `x0 - x18`.

* `state` Đây là trạng thái của **task** hiện đang chạy. Đối với các **task** chỉ đang thực hiện một số công việc trên CPU, trạng thái sẽ luôn là [TASK_RUNNING](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/sched.h#L15). Thực tế, đây là trạng thái duy nhất mà RPi OS sẽ hỗ trợ hiện tại. Tuy nhiên, sau này chúng ta sẽ phải thêm một số trạng thái bổ sung. Ví dụ, một **task** đang chờ **interrupt** nên được chuyển sang trạng thái khác, vì không có ý nghĩa gì khi đánh thức **task** trong khi **interrupt** cần thiết chưa xảy ra.

* `counter` Trường này được sử dụng để xác định **task** hiện tại đã chạy được bao lâu. `counter` giảm đi 1 mỗi **timer tick** và khi nó đạt 0 thì một **task** khác sẽ được lập lịch.

* `priority` Khi một **task** mới được lập lịch, `priority` của nó được sao chép vào `counter`. Bằng cách đặt **priority** của **task**, chúng ta có thể điều chỉnh lượng thời gian **processor** mà **task** nhận được so với các **task** khác.

* `preempt_count` Nếu trường này có giá trị khác không, nó là một chỉ báo rằng ngay lúc này **task** hiện tại đang thực thi một hàm quan trọng không được ngắt (ví dụ, nó đang chạy hàm **scheduling**). Nếu **timer tick** xảy ra vào thời điểm đó, nó sẽ bị bỏ qua và **rescheduling** không được kích hoạt.

Sau khi **kernel** khởi động, chỉ có một **task** đang chạy: cái chạy hàm [kernel_main](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/kernel.c#L19). Nó được gọi là "**init task**". Trước khi chức năng **scheduler** được kích hoạt, chúng ta phải điền `task_struct` tương ứng với **init task**. Điều này được thực hiện [tại đây](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/sched.h#L53).

Tất cả các **task** được lưu trữ trong **array** [task](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L7). **Array** này chỉ có 64 **slot** - đó là số lượng **task** đồng thời tối đa mà chúng ta có thể có trong RPi OS. Đây chắc chắn không phải là giải pháp tốt nhất cho OS sẵn sàng sản xuất, nhưng nó ổn cho mục tiêu của chúng ta.

Cũng có một biến rất quan trọng gọi là [current](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L6) luôn trỏ đến **task** hiện đang thực thi. Cả `current` và **array** `task` ban đầu đều được đặt để giữ một **pointer** đến **init task**. Cũng có một biến toàn cục gọi là [nr_tasks](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L8) - nó chứa số lượng **task** hiện đang chạy trong hệ thống.

Đó là tất cả các **structure** và biến toàn cục mà chúng ta sẽ sử dụng để triển khai chức năng **scheduler**. Trong mô tả của `task_struct` tôi đã đề cập ngắn gọn một số khía cạnh về cách **scheduling** hoạt động, vì không thể hiểu được ý nghĩa của một trường `task_struct` cụ thể mà không hiểu cách trường này được sử dụng. Bây giờ chúng ta sẽ xem xét thuật toán **scheduling** chi tiết hơn nhiều và chúng ta sẽ bắt đầu với hàm `kernel_main`.

### 4.2. Hàm `kernel_main`

Trước khi chúng ta đi sâu vào việc triển khai **scheduler**, tôi muốn nhanh chóng cho bạn thấy cách chúng ta sẽ chứng minh rằng **scheduler** thực sự hoạt động. Để hiểu điều này, bạn cần xem tệp [kernel.c](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/kernel.c). Hãy để tôi sao chép nội dung liên quan ở đây.

```c
void kernel_main(void)
{
    uart_init();
    init_printf(0, putc);
    irq_vector_init();
    timer_init();
    enable_interrupt_controller();
    enable_irq();

    int res = copy_process((unsigned long)&process, (unsigned long)"12345");
    if (res != 0) {
        printf("error while starting process 1");
        return;
    }
    res = copy_process((unsigned long)&process, (unsigned long)"abcde");
    if (res != 0) {
        printf("error while starting process 2");
        return;
    }

    while (1){
        schedule();
    }
}
```

Có một số điều quan trọng về đoạn code này:

1. Hàm mới [copy_process](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/fork.c#L5) được giới thiệu. `copy_process` nhận 2 tham số: một hàm để thực thi trong **thread** mới và một tham số cần được truyền cho hàm này. `copy_process` cấp phát một `task_struct` mới và làm cho nó có sẵn cho **scheduler**.

2. Một hàm mới khác cho chúng ta được gọi là [schedule](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L21). Đây là hàm **scheduler** cốt lõi: nó kiểm tra xem có **task** mới nào cần **preempt** **task** hiện tại không. Một **task** có thể tự nguyện gọi `schedule` nếu nó không có công việc gì để làm vào lúc này. `schedule` cũng được gọi từ **timer interrupt handler**.

Chúng ta gọi `copy_process` 2 lần, mỗi lần truyền một **pointer** đến hàm [process](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/kernel.c#L9) làm tham số đầu tiên. Hàm `process` rất đơn giản:

```c
void process(char *array)
{
    while (1){
        for (int i = 0; i < 5; i++){
            uart_send(array[i]);
            delay(100000);
        }
    }
}
```

Nó chỉ tiếp tục in lên màn hình các ký tự từ **array** được truyền làm tham số. Lần đầu tiên nó được gọi với tham số "12345" và lần thứ hai tham số là "abcde". Nếu việc triển khai **scheduler** của chúng ta đúng, chúng ta sẽ thấy trên màn hình đầu ra hỗn hợp từ cả hai **thread**.

### 4.3. Cấp phát bộ nhớ (Memory allocation)

Mỗi **task** trong hệ thống phải có **stack** riêng biệt. Đó là lý do tại sao khi tạo một **task** mới, chúng ta phải có cách để cấp phát bộ nhớ. Hiện tại, **memory allocator** của chúng ta cực kỳ nguyên thủy. (Việc triển khai có thể được tìm thấy trong tệp [mm.c](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/mm.c))

```c
static unsigned short mem_map [ PAGING_PAGES ] = {0,};

unsigned long get_free_page()
{
    for (int i = 0; i < PAGING_PAGES; i++){
        if (mem_map[i] == 0){
            mem_map[i] = 1;
            return LOW_MEMORY + i*PAGE_SIZE;
        }
    }
    return 0;
}

void free_page(unsigned long p){
    mem_map[(p - LOW_MEMORY) / PAGE_SIZE] = 0;
}
```

**Allocator** chỉ có thể làm việc với các **memory page** (mỗi **page** có kích thước 4 KB). Có một **array** gọi là `mem_map` mà đối với mỗi **page** trong hệ thống giữ trạng thái của nó: liệu nó đã được cấp phát hay còn trống. Bất cứ khi nào chúng ta cần cấp phát một **page** mới, chúng ta chỉ lặp qua **array** này và trả về **page** trống đầu tiên. Việc triển khai này dựa trên 2 giả định:

1. Chúng ta biết tổng lượng bộ nhớ trong hệ thống. Nó là `1 GB - 1 MB` (megabyte cuối cùng của bộ nhớ được dành riêng cho **device register**). Giá trị này được lưu trữ trong hằng số [HIGH_MEMORY](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/mm.h#L14).

2. 4 MB đầu tiên của bộ nhớ được dành riêng cho **kernel image** và **init task stack**. Giá trị này được lưu trữ trong hằng số [LOW_MEMORY](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/mm.h#L13). Tất cả việc cấp phát bộ nhớ bắt đầu ngay sau điểm này.

### 4.4. Tạo một task mới

Việc cấp phát **task** mới được triển khai trong hàm [copy_process](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/fork.c#L5).

```c
int copy_process(unsigned long fn, unsigned long arg)
{
    preempt_disable();
    struct task_struct *p;

    p = (struct task_struct *) get_free_page();
    if (!p)
        return 1;
    p->priority = current->priority;
    p->state = TASK_RUNNING;
    p->counter = p->priority;
    p->preempt_count = 1; //disable preemtion until schedule_tail

    p->cpu_context.x19 = fn;
    p->cpu_context.x20 = arg;
    p->cpu_context.pc = (unsigned long)ret_from_fork;
    p->cpu_context.sp = (unsigned long)p + THREAD_SIZE;
    int pid = nr_tasks++;
    task[pid] = p;
    preempt_enable();
    return 0;
}
```

Bây giờ, chúng ta sẽ xem xét nó chi tiết.

```c
    preempt_disable();
    struct task_struct *p;
```

Hàm bắt đầu bằng việc vô hiệu hóa **preemption** và cấp phát một **pointer** cho **task** mới. **Preemption** bị vô hiệu hóa vì chúng ta không muốn bị **reschedule** sang một **task** khác ở giữa hàm `copy_process`.

```c
    p = (struct task_struct *) get_free_page();
    if (!p)
        return 1;
```

Tiếp theo, một **page** mới được cấp phát. Ở phía dưới của **page** này, chúng ta đặt `task_struct` cho **task** mới được tạo. Phần còn lại của **page** này sẽ được sử dụng làm **task stack**.

```c
    p->priority = current->priority;
    p->state = TASK_RUNNING;
    p->counter = p->priority;
    p->preempt_count = 1; //disable preemtion until schedule_tail
```

Sau khi `task_struct` được cấp phát, chúng ta có thể khởi tạo các thuộc tính của nó. **Priority** và **counter** ban đầu được đặt dựa trên **priority** của **task** hiện tại. **State** được đặt thành `TASK_RUNNING`, cho biết rằng **task** mới đã sẵn sàng để được khởi động. `preempt_count` được đặt thành 1, có nghĩa là sau khi **task** được thực thi, nó không nên bị **reschedule** cho đến khi hoàn thành một số công việc khởi tạo.

```c
    p->cpu_context.x19 = fn;
    p->cpu_context.x20 = arg;
    p->cpu_context.pc = (unsigned long)ret_from_fork;
    p->cpu_context.sp = (unsigned long)p + THREAD_SIZE;
```

Đây là phần quan trọng nhất của hàm. Ở đây `cpu_context` được khởi tạo. **Stack pointer** được đặt ở đỉnh của **memory page** mới được cấp phát. `pc` được đặt thành hàm [ret_from_fork](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L146), và chúng ta cần xem hàm này ngay bây giờ để hiểu tại sao phần còn lại của các **register** `cpu_context` được khởi tạo theo cách như vậy.

```assembly
.globl ret_from_fork
ret_from_fork:
    bl    schedule_tail
    mov    x0, x20
    blr    x19         //should never return
```

Như bạn có thể thấy `ret_from_fork` đầu tiên gọi [schedule_tail](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L65), chỉ đơn giản là kích hoạt **preemption**, và sau đó nó gọi hàm được lưu trữ trong **register** `x19` với tham số được lưu trữ trong `x20`. `x19` và `x20` được khôi phục từ `cpu_context` ngay trước khi hàm `ret_from_fork` được gọi.

Bây giờ, hãy quay lại `copy_process`.

```c
    int pid = nr_tasks++;
    task[pid] = p;
    preempt_enable();
    return 0;
```

Cuối cùng, `copy_process` thêm **task** mới được tạo vào **array** `task` và kích hoạt **preemption** cho **task** hiện tại.

Một điều quan trọng cần hiểu về hàm `copy_process` là sau khi nó hoàn thành thực thi, không có **context switch** nào xảy ra. Hàm chỉ chuẩn bị `task_struct` mới và thêm nó vào **array** `task` — **task** này sẽ chỉ được thực thi sau khi hàm `schedule` được gọi.

### 4.5. Ai gọi `schedule`?

Trước khi chúng ta đi vào chi tiết của hàm `schedule`, hãy tìm hiểu trước cách `schedule` được gọi. Có 2 kịch bản:

1. Khi một **task** không có gì để làm ngay bây giờ, nhưng nó vẫn không thể bị chấm dứt, nó có thể gọi `schedule` một cách tự nguyện. Đó là điều mà hàm `kernel_main` làm.

2. `schedule` cũng được gọi thường xuyên từ [timer interrupt handler](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/timer.c#L21).

Bây giờ hãy xem hàm [timer_tick](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L70), được gọi từ **timer interrupt**.

```c
void timer_tick()
{
    --current->counter;
    if (current->counter>0 || current->preempt_count >0) {
        return;
    }
    current->counter=0;
    enable_irq();
    _schedule();
    disable_irq();
}
```

Trước tiên, nó giảm **counter** của **task** hiện tại. Nếu **counter** lớn hơn 0 hoặc **preemption** hiện đang bị vô hiệu hóa thì hàm trả về, nhưng nếu không thì `schedule` được gọi với **interrupt** được kích hoạt. (Chúng ta đang ở trong một **interrupt handler**, và **interrupt** bị vô hiệu hóa theo mặc định) Chúng ta sẽ thấy tại sao **interrupt** phải được kích hoạt trong quá trình thực thi **scheduler** trong phần tiếp theo.

### 4.6. Thuật toán Scheduling

Cuối cùng, chúng ta đã sẵn sàng để xem thuật toán **scheduler**. Tôi đã sao chép gần như chính xác thuật toán này từ phiên bản đầu tiên của **Linux kernel**. Bạn có thể tìm thấy phiên bản gốc [tại đây](https://github.com/zavg/linux-0.01/blob/master/kernel/sched.c#L68).

```c
void _schedule(void)
{
    preempt_disable();
    int next,c;
    struct task_struct * p;
    while (1) {
        c = -1;
        next = 0;
        for (int i = 0; i < NR_TASKS; i++){
            p = task[i];
            if (p && p->state == TASK_RUNNING && p->counter > c) {
                c = p->counter;
                next = i;
            }
        }
        if (c) {
            break;
        }
        for (int i = 0; i < NR_TASKS; i++) {
            p = task[i];
            if (p) {
                p->counter = (p->counter >> 1) + p->priority;
            }
        }
    }
    switch_to(task[next]);
    preempt_enable();
}
```

Thuật toán hoạt động như sau:

* Vòng lặp `for` bên trong đầu tiên lặp qua tất cả các **task** và cố gắng tìm một **task** ở trạng thái `TASK_RUNNING` với **counter** tối đa. Nếu **task** như vậy được tìm thấy và **counter** của nó lớn hơn 0, chúng ta ngay lập tức thoát khỏi vòng lặp `while` bên ngoài và chuyển sang **task** này. Nếu không tìm thấy **task** như vậy, điều này có nghĩa là không có **task** nào ở trạng thái `TASK_RUNNING` hiện tại hoặc tất cả các **task** như vậy đều có **counter** bằng 0. Trong một OS thực tế, trường hợp đầu tiên có thể xảy ra, ví dụ, khi tất cả các **task** đang chờ một **interrupt**. Trong trường hợp này, vòng lặp `for` lồng nhau thứ hai được thực thi. Đối với mỗi **task** (bất kể nó ở trạng thái nào), vòng lặp này tăng **counter** của nó. Việc tăng **counter** được thực hiện theo cách rất thông minh:

    1. **Task** trải qua càng nhiều lần lặp của vòng lặp `for` thứ hai, **counter** của nó sẽ được tăng càng nhiều.
    2. **Counter** của một **task** không bao giờ có thể lớn hơn `2 * priority`.

* Sau đó quá trình được lặp lại. Nếu có ít nhất một **task** ở trạng thái `TASK_RUNNING`, lần lặp thứ hai của vòng lặp `while` bên ngoài sẽ là lần cuối cùng vì sau lần lặp đầu tiên tất cả các **counter** đã khác không. Tuy nhiên, nếu không có **task** `TASK_RUNNING` nào ở đó, quá trình được lặp lại liên tục cho đến khi một số **task** sẽ chuyển sang trạng thái `TASK_RUNNING`. Nhưng nếu chúng ta đang chạy trên một CPU duy nhất, làm thế nào mà trạng thái **task** có thể thay đổi trong khi vòng lặp này đang chạy? Câu trả lời là nếu một **task** nào đó đang chờ một **interrupt**, **interrupt** này có thể xảy ra trong khi hàm `schedule` đang được thực thi và **interrupt handler** có thể thay đổi trạng thái của **task**. Điều này thực sự giải thích tại sao **interrupt** phải được kích hoạt trong quá trình thực thi `schedule`. Điều này cũng chứng minh sự khác biệt quan trọng giữa việc vô hiệu hóa **interrupt** và vô hiệu hóa **preemption**. `schedule` vô hiệu hóa **preemption** trong suốt thời gian thực thi toàn bộ hàm. Điều này đảm bảo rằng `schedule` lồng nhau sẽ không được gọi trong khi chúng ta đang ở giữa việc thực thi hàm gốc. Tuy nhiên, **interrupt** có thể hợp pháp xảy ra trong quá trình thực thi hàm `schedule`.

Tôi đã dành rất nhiều sự chú ý đến tình huống mà một **task** nào đó đang chờ một **interrupt**, mặc dù chức năng này chưa được triển khai trong RPi OS. Nhưng tôi vẫn coi việc hiểu trường hợp này là cần thiết vì nó là một phần của thuật toán **scheduler** cốt lõi và chức năng tương tự sẽ được thêm vào sau này.

### 4.7. Chuyển đổi task (Switching tasks)

Sau khi **task** ở trạng thái `TASK_RUNNING` với **counter** khác không được tìm thấy, hàm [switch_to](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L56) được gọi. Nó trông như thế này:

```c
void switch_to(struct task_struct * next)
{
    if (current == next)
        return;
    struct task_struct * prev = current;
    current = next;
    cpu_switch_to(prev, next);
}
```

Ở đây chúng ta kiểm tra rằng **process** tiếp theo không giống với **process** hiện tại, và nếu không, biến `current` được cập nhật. Công việc thực tế được chuyển hướng đến hàm [cpu_switch_to](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.S).

```assembly
.globl cpu_switch_to
cpu_switch_to:
    mov    x10, #THREAD_CPU_CONTEXT
    add    x8, x0, x10
    mov    x9, sp
    stp    x19, x20, [x8], #16        // store callee-saved registers
    stp    x21, x22, [x8], #16
    stp    x23, x24, [x8], #16
    stp    x25, x26, [x8], #16
    stp    x27, x28, [x8], #16
    stp    x29, x9, [x8], #16
    str    x30, [x8]
    add    x8, x1, x10
    ldp    x19, x20, [x8], #16        // restore callee-saved registers
    ldp    x21, x22, [x8], #16
    ldp    x23, x24, [x8], #16
    ldp    x25, x26, [x8], #16
    ldp    x27, x28, [x8], #16
    ldp    x29, x9, [x8], #16
    ldr    x30, [x8]
    mov    sp, x9
    ret
```

Đây là nơi **context switch** thực sự xảy ra. Hãy xem xét từng dòng một.

```assembly
    mov    x10, #THREAD_CPU_CONTEXT
    add    x8, x0, x10
```

Hằng số `THREAD_CPU_CONTEXT` chứa **offset** của **structure** `cpu_context` trong `task_struct`. `x0` chứa một **pointer** đến tham số đầu tiên, đó là `task_struct` hiện tại (bằng hiện tại ở đây tôi có ý là cái mà chúng ta đang chuyển từ). Sau khi 2 dòng được sao chép được thực thi, `x8` sẽ chứa một **pointer** đến `cpu_context` hiện tại.

```assembly
    mov    x9, sp
    stp    x19, x20, [x8], #16        // store callee-saved registers
    stp    x21, x22, [x8], #16
    stp    x23, x24, [x8], #16
    stp    x25, x26, [x8], #16
    stp    x27, x28, [x8], #16
    stp    x29, x9, [x8], #16
    str    x30, [x8]
```

Tiếp theo tất cả các **callee-saved register** được lưu trữ theo thứ tự mà chúng được định nghĩa trong **structure** `cpu_context`. `x30`, là **link register** và chứa địa chỉ trả về của hàm, được lưu trữ dưới dạng `pc`, **stack pointer** hiện tại được lưu dưới dạng `sp` và `x29` được lưu dưới dạng `fp` (**frame pointer**).

```assembly
    add    x8, x1, x10
```

Bây giờ `x10` chứa một **offset** của **structure** `cpu_context` bên trong `task_struct`, `x1` là một **pointer** đến `task_struct` tiếp theo, vì vậy `x8` sẽ chứa một **pointer** đến `cpu_context` tiếp theo.

```assembly
    ldp    x19, x20, [x8], #16        // restore callee-saved registers
    ldp    x21, x22, [x8], #16
    ldp    x23, x24, [x8], #16
    ldp    x25, x26, [x8], #16
    ldp    x27, x28, [x8], #16
    ldp    x29, x9, [x8], #16
    ldr    x30, [x8]
    mov    sp, x9
```

Các **callee saved register** được khôi phục từ `cpu_context` tiếp theo.

```assembly
    ret
```

Hàm trả về vị trí được trỏ bởi **link register** (`x30`). Nếu chúng ta đang chuyển sang một **task** nào đó lần đầu tiên, đây sẽ là hàm [ret_from_fork](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L148). Trong tất cả các trường hợp khác, đây sẽ là vị trí được lưu trước đó trong `cpu_context` bởi hàm `cpu_switch_to`.

### 4.8. Scheduling hoạt động như thế nào với exception entry/exit?

Trong bài học trước, chúng ta đã thấy cách các **macro** [kernel_entry](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L17) và [kernel_exit](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L4) được sử dụng để lưu và khôi phục trạng thái **processor**. Sau khi **scheduler** được giới thiệu, một vấn đề mới xuất hiện: bây giờ việc vào một **interrupt** với tư cách là một **task** và rời khỏi nó với tư cách là một **task** khác trở nên hoàn toàn hợp pháp. Đây là một vấn đề, vì lệnh `eret` mà chúng ta đang sử dụng để trả về từ các **interrupt**, dựa vào thực tế là địa chỉ trả về phải được lưu trữ trong **register** `elr_el1` và trạng thái **processor** trong **register** `spsr_el1`. Vì vậy, nếu chúng ta muốn chuyển đổi **task** trong khi xử lý một **interrupt**, chúng ta phải lưu và khôi phục 2 **register** đó cùng với tất cả các **general purpose register** khác. Code thực hiện điều này rất đơn giản, bạn có thể tìm thấy phần lưu [tại đây](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L35) và khôi phục [tại đây](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L46).

### 4.9. Theo dõi trạng thái hệ thống trong quá trình context switch

Chúng ta đã xem xét tất cả **source code** liên quan đến **context switch**. Tuy nhiên, code đó chứa rất nhiều tương tác bất đồng bộ khiến việc hiểu đầy đủ cách trạng thái của toàn bộ hệ thống thay đổi theo thời gian trở nên khó khăn. Trong phần này tôi muốn làm cho nhiệm vụ này dễ dàng hơn cho bạn: tôi muốn mô tả chuỗi các sự kiện xảy ra từ khi hệ thống khởi động đến thời điểm **context switch** thứ hai. Đối với mỗi sự kiện như vậy, tôi cũng sẽ bao gồm một sơ đồ biểu diễn trạng thái của bộ nhớ tại thời điểm của sự kiện. Tôi hy vọng rằng cách biểu diễn như vậy sẽ giúp bạn hiểu sâu về cách **scheduler** hoạt động. Vậy, hãy bắt đầu!

1. **Kernel** được khởi tạo và hàm [kernel_main](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/kernel.c#L19) được thực thi. **Stack** ban đầu được cấu hình để bắt đầu tại [LOW_MEMORY](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/include/mm.h#L13), là 4 MB.
    ```
             0 +------------------+
               | kernel image     |
               |------------------|
               |                  |
               |------------------|
               | init task stack  |
    0x00400000 +------------------+
               |                  |
               |                  |
    0x3F000000 +------------------+
               | device registers |
    0x40000000 +------------------+
    ```

2. `kernel_main` gọi [copy_process](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/fork.c#L5) lần đầu tiên. **Memory page** 4 KB mới được cấp phát, và `task_struct` được đặt ở phía dưới của **page** này. (Sau này tôi sẽ gọi **task** được tạo tại thời điểm này là "**task 1**")
    ```
             0 +------------------+
               | kernel image     |
               |------------------|
               |                  |
               |------------------|
               | init task stack  |
    0x00400000 +------------------+
               | task_struct 1    |
               |------------------|
               |                  |
    0x00401000 +------------------+
               |                  |
               |                  |
    0x3F000000 +------------------+
               | device registers |
    0x40000000 +------------------+
    ```

3. `kernel_main` gọi `copy_process` lần thứ hai và quá trình tương tự lặp lại. **Task 2** được tạo và thêm vào danh sách **task**.
    ```
             0 +------------------+
               | kernel image     |
               |------------------|
               |                  |
               |------------------|
               | init task stack  |
    0x00400000 +------------------+
               | task_struct 1    |
               |------------------|
               |                  |
    0x00401000 +------------------+
               | task_struct 2    |
               |------------------|
               |                  |
    0x00402000 +------------------+
               |                  |
               |                  |
    0x3F000000 +------------------+
               | device registers |
    0x40000000 +------------------+
    ```

4. `kernel_main` tự nguyện gọi hàm [schedule](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L21) và nó quyết định chạy **task 1**.

5. [cpu_switch_to](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.S#L4) lưu các **callee-saved register** trong `cpu_context` của **init task**, nằm bên trong **kernel image**.

6. `cpu_switch_to` khôi phục các **callee-saved register** từ `cpu_context` của **task 1**. `sp` bây giờ trỏ đến `0x00401000`, **link register** đến hàm [ret_from_fork](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L146), `x19` chứa một **pointer** đến hàm [process](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/kernel.c#L9) và `x20` một **pointer** đến chuỗi "12345", nằm ở đâu đó trong **kernel image**.

7. `cpu_switch_to` gọi lệnh `ret`, nhảy đến hàm `ret_from_fork`.

8. `ret_from_fork` đọc các **register** `x19` và `x20` và gọi hàm `process` với tham số "12345". Sau khi hàm `process` bắt đầu thực thi, **stack** của nó bắt đầu phát triển.
    ```
             0 +------------------+
               | kernel image     |
               |------------------|
               |                  |
               |------------------|
               | init task stack  |
    0x00400000 +------------------+
               | task_struct 1    |
               |------------------|
               |                  |
               |------------------|
               | task 1 stack     |
    0x00401000 +------------------+
               | task_struct 2    |
               |------------------|
               |                  |
    0x00402000 +------------------+
               |                  |
               |                  |
    0x3F000000 +------------------+
               | device registers |
    0x40000000 +------------------+
    ```

9. Một **timer interrupt** xảy ra. **Macro** [kernel_entry](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/entry.S#L17) lưu tất cả các **general purpose register** + `elr_el1` và `spsr_el1` vào phía dưới của **task 1 stack**.
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 saved registers |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

10. `schedule` được gọi và nó quyết định chạy **task 2**. Nhưng chúng ta vẫn đang chạy **task 1** và **stack** của nó tiếp tục phát triển bên dưới vùng **task 1 saved registers**. Trên sơ đồ, phần này của **stack** được đánh dấu là (int), có nghĩa là "**interrupt stack**"
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 stack (int)     |
               |------------------------|
               | task 1 saved registers |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

11. `cpu_switch_to` chạy **task 2**. Để làm điều này, nó thực thi chính xác cùng một chuỗi các bước mà nó thực hiện cho **task 1**. **Task 2** bắt đầu thực thi và **stack** của nó phát triển. Lưu ý rằng chúng ta không trả về từ một **interrupt** tại thời điểm này, nhưng điều này ổn vì **interrupt** bây giờ được kích hoạt (**interrupt** đã được kích hoạt trước đó trong [timer_tick](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L70) trước khi `schedule` được gọi)
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 stack (int)     |
               |------------------------|
               | task 1 saved registers |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
               |------------------------|
               | task 2 stack           |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

12. Một **timer interrupt** khác xảy ra và `kernel_entry` lưu tất cả các **general purpose register** + `elr_el1` và `spsr_el1` ở phía dưới của **task 2 stack**. **Task 2 interrupt stack** bắt đầu phát triển.
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 stack (int)     |
               |------------------------|
               | task 1 saved registers |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
               |------------------------|
               | task 2 stack (int)     |
               |------------------------|
               | task 2 saved registers |
               |------------------------|
               | task 2 stack           |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

13. `schedule` được gọi. Nó quan sát thấy rằng tất cả các **task** đều có **counter** được đặt thành 0 và đặt **counter** thành **priority** của các **task**.

14. `schedule` chọn **init task** để chạy. (Điều này là do tất cả các **task** bây giờ đều có **counter** được đặt thành 1 và **init task** là đầu tiên trong danh sách). Nhưng thực tế, việc `schedule` chọn **task 1** hoặc **task 2** tại thời điểm này sẽ hoàn toàn hợp pháp, vì **counter** của chúng có giá trị bằng nhau. Chúng ta quan tâm hơn đến trường hợp khi **task 1** được chọn nên bây giờ hãy giả sử rằng đây là điều đã xảy ra.

15. `cpu_switch_to` được gọi và nó khôi phục các **callee-saved register** đã lưu trước đó từ `cpu_context` của **task 1**. **Link register** bây giờ trỏ [tại đây](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L63) vì đây là nơi mà `cpu_switch_to` được gọi lần cuối khi **task 1** được thực thi. `sp` trỏ đến phía dưới của **task 1 interrupt stack**.

16. Hàm `timer_tick` tiếp tục thực thi, bắt đầu từ [dòng này](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L79). Nó vô hiệu hóa **interrupt** và cuối cùng [kernel_exit](https://github.com/kiennql/raspberry-pi-os/blob/master/src/lesson04/src/sched.c#L79) được thực thi. Vào thời điểm `kernel_exit` được gọi, **task 1 interrupt stack** được thu gọn về 0.
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 saved registers |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
               |------------------------|
               | task 2 stack (int)     |
               |------------------------|
               | task 2 saved registers |
               |------------------------|
               | task 2 stack           |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

17. `kernel_exit` khôi phục tất cả các **general purpose register** cũng như `elr_el1` và `spsr_el1`. `elr_el1` bây giờ trỏ đến đâu đó ở giữa hàm `process`. `sp` trỏ đến phía dưới của **task 1 stack**.
    ```
             0 +------------------------+
               | kernel image           |
               |------------------------|
               |                        |
               |------------------------|
               | init task stack        |
    0x00400000 +------------------------+
               | task_struct 1          |
               |------------------------|
               |                        |
               |------------------------|
               | task 1 stack           |
    0x00401000 +------------------------+
               | task_struct 2          |
               |------------------------|
               |                        |
               |------------------------|
               | task 2 stack (int)     |
               |------------------------|
               | task 2 saved registers |
               |------------------------|
               | task 2 stack           |
    0x00402000 +------------------------+
               |                        |
               |                        |
    0x3F000000 +------------------------+
               | device registers       |
    0x40000000 +------------------------+
    ```

18. `kernel_exit` thực thi lệnh `eret` sử dụng **register** `elr_el1` để nhảy trở lại hàm `process`. **Task 1** tiếp tục thực thi bình thường.

Chuỗi các bước được mô tả ở trên rất quan trọng — cá nhân tôi coi nó là một trong những điều quan trọng nhất trong toàn bộ hướng dẫn. Nếu bạn gặp khó khăn trong việc hiểu nó, tôi có thể khuyên bạn làm bài tập số 1 từ bài học này.

### 4.10. Kết luận

Chúng ta đã hoàn thành với **scheduling**, nhưng ngay bây giờ **kernel** của chúng ta chỉ có thể quản lý các **kernel thread**: chúng được thực thi ở EL1 và có thể truy cập trực tiếp bất kỳ hàm hoặc dữ liệu **kernel** nào. Trong 2 bài học tiếp theo, chúng ta sẽ khắc phục điều này và giới thiệu **system call** và **virtual memory**.