---
title: "IO: Thuật toán IO Readahead trong nhân Linux (Phần 1)"
author: kiennql
date: 2025-06-20 15:56:00 +0700
categories: [io, paper, kernel]
tags: [linux kernel, io optimization, readahead, performance, algorithm, file system, vfs, ext4]
description: "Phân tích thuật toán on-demand readahead trong Linux kernel, tối ưu hiệu suất IO với sequential access và cache management."
math: true
mermaid: true
render_with_liquid: false
---

## 1. Giới thiệu

Chìa khóa tối ưu hóa hiệu suất nằm ở việc giải quyết các bottleneck hiệu suất, và IO luôn là một trong những bottleneck khó giải quyết nhất. Bài viết này chủ yếu dựa trên paper của tác giả thuật toán, mô tả thuật toán on-demand readahead cho các thao tác đọc trong Linux kernel.

## 2. Các chiến lược tối ưu IO cơ bản

Trước khi bắt đầu nói về thuật toán prefetch, chúng ta cần xem qua các chiến lược tối ưu IO thông thường:

1. **Tránh IO**: Các chiến lược cache thông thường thực chất là tránh IO (external storage), không có IO thì không có vấn đề IO =)))
2. **Sequential IO**: Rõ ràng, truy cập tuần tự nhanh hơn truy cập ngẫu nhiên. Nếu không thể thực hiện hoàn toàn tuần tự với nhiều IO load, thì tăng kích thước IO để giảm seek time và tăng throughput
3. **Asynchronous IO**: CPU và disk hoạt động đồng thời, tải trước dữ liệu cần thiết vào memory
4. **Parallel IO**: Thực hiện parallel IO thông qua RAID

Tuy nhiên, đối với một ứng dụng thông thường, hành vi IO của nó là nhỏ và serial: không thể async, không thể parallel, cũng không thể tăng throughput. Đối với những hành vi này, chúng ta cần preread dữ liệu và delayed write (mối quan hệ của chúng không orthogonal, delayed write như write back sẽ không được thảo luận riêng).

Lý do cần thiết rất đơn giản: thông qua preread để đáp ứng các phương pháp tối ưu 1, 2, 3 ở trên.

## 3. Hai loại preread

Preread có thể chia thành:
- **Notification-based preread**
- **Heuristic preread**

Notification-based preread cần sự hợp tác của user, như `posix_fadvise`, không bàn ở blog này. Heuristic hoàn toàn transparent với user, khó khăn nằm ở yêu cầu thuật toán cao, cũng là nội dung cụ thể sẽ được giới thiệu sau.

## 4. Hai chiến lược preread heuristic

Ngoài direct read, Linux sẽ thực hiện một số chiến lược preread cho read interface:
- **`readaround`** đại diện bởi `mmap`
- **`readahead`** đại diện bởi `read`

Sự khác biệt chiến lược của chúng nằm ở việc tập trung hơn vào random read hay sequential read (hoặc là universal, bất kỳ loại read operation nào).

Có thể mô tả quá trình `readaround` trong `mmap` một cách đơn giản:
1. `mmap` chỉ cung cấp `vma` và mapping relationship tương ứng trả về cho user space
2. Khi user space trigger page fault tương ứng, kiểm tra xem có cache không, nếu không có thì prefetch một số page với page hiện tại làm trung tâm

Còn `read-ahead` phức tạp hơn nhiều và khó implement hơn.

## 5. Độ phức tạp của readahead

`readahead` so với `readaround` là nó sẽ tập trung hơn vào hiệu suất sequential read, nhưng cũng cần xem xét tình huống dưới nhiều IO load khác nhau. Vì nó nằm ở tầng `generic read` dưới `VFS`, các interface như `read/pread/readv/sendfile` đều sẽ thống nhất vào `generic read`, tình huống phải đối mặt cũng phức tạp hơn:

- Sequential access phức tạp hơn, có thể có non-aligned read, retry read, interleaved read
- Sequential access có thể không cần prefetch, có thể có tình huống readahead cache hit
- Page được prefetch có thể bị reclaim trước khi access, tức là hiện tượng readahead thrashing

Để implement `readahead`, cần xử lý các tình huống phức tạp trên.

## 6. On-demand readahead

Hiện tại kernel cung cấp framework thuật toán on-demand readahead ở generic block layer. Theo nghĩa đen là **readahead theo nhu cầu**.

Tôi nghĩ chia thành 3 module để mô tả toàn bộ framework khá phù hợp:
1. Cấu trúc dữ liệu cần thiết
2. Entry point của thuật toán  
3. Implementation của thuật toán

*(Lưu ý: Mô tả thuật toán dưới đây dựa trên implementation Linux 4.18.20)*

### 6.1. Trạng thái readahead

Cấu trúc dữ liệu `readahead` cần thiết nằm trong `/include/linux/fs.h`:

```c
/* 
 * Track a single file's readahead state 
 */
struct file_ra_state {
        pgoff_t start;                  /* where readahead started */
        unsigned int size;              /* # of readahead pages */
        unsigned int async_size;        /* do asynchronous readahead when
                                           there are only # of pages ahead */
        unsigned int ra_pages;	        /* Maximum readahead window */
        unsigned int mmap_miss;         /* Cache miss stat for mmap accesses */
        loff_t prev_pos;                /* Cache last read() position */
};
```

Cần chú ý các member `start` / `size` / `async_size`, chúng duy trì ahead window thông qua không gian \\(O(1)\\).

Theo hình của tác giả để mô tả:

```
/* 
 * The fields in struct file_ra_state represent the most-recently-executed 
 * readahead attempt: 
 * 
 *                        |<----- async_size ---------|
 *     |------------------- size -------------------->|
 *     |==================#===========================|
 *     ^start             ^page marked with PG_readahead 
 */
```

- `start` và `size` tạo thành readahead window, ghi lại vị trí `start` và kích thước liên tục `size` của request readahead gần nhất
- `async_size` là readahead advance amount, biểu thị khi còn lại `async_size` page chưa access thì khởi động readahead tiếp theo

Sau này sẽ mô tả việc thực hiện IO pipeline operation thông qua ahead window.

### 6.2. Entry point tầng generic

Lấy `read` / `ext4` làm ví dụ trace, qua:

```c
// read -> ksys_read -> vfs_read -> __vfs_read -> file_operations->read_iter
// -> new_sync_read -> call_read_iter -> generic_file_read_iter
// -> generic_file_buffered_read

static ssize_t generic_file_buffered_read(struct kiocb *iocb,
                struct iov_iter *iter, ssize_t written)
{
        struct file *filp = iocb->ki_filp;
        struct address_space *mapping = filp->f_mapping;
        struct inode *inode = mapping->host;
        struct file_ra_state *ra = &filp->f_ra;
        loff_t *ppos = &iocb->ki_pos;
        pgoff_t index;
        pgoff_t last_index;
        pgoff_t prev_index;
        unsigned long offset;      /* offset into pagecache page */
        unsigned int prev_offset;
        int error = 0;

        if (unlikely(*ppos >= inode->i_sb->s_maxbytes))
                return 0;

        iov_iter_truncate(iter, inode->i_sb->s_maxbytes);

        index = *ppos >> PAGE_SHIFT;
        prev_index = ra->prev_pos >> PAGE_SHIFT;
        prev_offset = ra->prev_pos & (PAGE_SIZE-1);
        last_index = (*ppos + iter->count + PAGE_SIZE-1) >> PAGE_SHIFT;
        offset = *ppos & ~PAGE_MASK;

        for (;;) {
                struct page *page;
                pgoff_t end_index;
                loff_t isize;
                unsigned long nr, ret;

                cond_resched();
find_page:
                if (fatal_signal_pending(current)) {
                        error = -EINTR;
                        goto out;
                }

                page = find_get_page(mapping, index);
                if (!page) {
                        if (iocb->ki_flags & IOCB_NOWAIT)
                                goto would_block;
                        page_cache_sync_readahead(mapping, // ⭐
                                        ra, filp,
                                        index, last_index - index);
                        page = find_get_page(mapping, index);
                        if (unlikely(page == NULL))
                                goto no_cached_page;
                }
                if (PageReadahead(page)) {
                        page_cache_async_readahead(mapping, // ⭐
                                        ra, filp, page,
                                        index, last_index - index);
                }
        // ...
```

Trong đó `page_cache_sync_readahead` và `page_cache_async_readahead` là entry point của readahead.

Có thể thấy, điều kiện vào readahead có hai loại:
- Không tìm thấy page cache trong radix tree
- Tìm thấy page cache và page đó được đánh dấu `PG_Readahead`

### 6.3. Quy trình đơn giản

Tôi cố gắng mô tả quá trình readahead của một read request sequential hoàn toàn một cách đơn giản.

Ký hiệu `| =` là single page, `#` là page được đánh dấu `PG_readahead`, `^` là `ra->start`, `~` là `offset` hiện tại đang access.

Khi sequential read một file/page chưa từng access, và bắt đầu từ `offset = 0, request_size = 1` (đơn vị page), sẽ trigger **synchronous readahead**:

```
|~
```

Do chưa từng access, thông tin tuple trong `file_ra_state` (viết tắt `ra`) là:

```
.start = 0
.size = 0  
.async_size = 0
.prev_pos = -1
```

Quy trình readahead sẽ dựa trên `prev_pos` để tạo readahead window, tính `size` và `async_size`, và đánh dấu `PG_readahead` tại `offset = #`:

```
|===#======|
^~
```

Trong đó `#` đến `|` bên phải là phạm vi `async_size`, tổng cộng đọc vào `size` page.

Giả sử lần sequential read tiếp theo bắt đầu từ page (`offset`) sau `^` và trước `#`, thì không trigger readahead:

```
|===#======|
^  ~
```

Tiếp tục sequential read đến `offset = #`, sẽ phát ra **asynchronous readahead** tiếp theo:

```
|==========|#======================|
    ~       ^
```

Do trong quá trình này `~` đang ở page đã prefetch, nên lần readahead này hoàn toàn async giữa CPU và IO.

Có thể thấy, readahead window là request readahead lần này, cũng là cơ sở phán đoán timing readahead lần tiếp theo.

Sự tăng trưởng của window thường theo dạng exponential, về sau sẽ tiếp tục thực hiện readahead theo **thiếu page cache** hoặc **trigger marked page**.

### 6.4. Quy trình tổng thể

1. Page cache miss, đi đến bước 3
2. Hoặc duyệt đến page `PG_readahead`, nếu điều kiện không cho phép thì thoát, ngược lại đi đến bước 3
3. Cố gắng xử lý initial sequential read hoặc head read, thành công thì đi đến bước 9
4. Cố gắng xử lý continuous sequential read, thành công thì đi đến bước 9
5. Cố gắng xử lý interleaved sequential read, thành công thì đi đến bước 9
6. Cố gắng xử lý oversized read, thành công đi đến bước 9
7. Cố gắng xử lý historical cache context, thành công thì đi đến bước 9
8. Xác nhận random read, submit IO, thoát
9. Cố gắng cho possible synchronous readahead hoặc context merge request
10. Đánh dấu một page nào đó là `PG_readahead`, submit IO

Có thể thấy, `read ahead` rất chú trọng vấn đề sequential read (bước xử lý nhiều nhất, cũng là priority phán đoán cao nhất) nhưng đồng thời cũng xem xét tình huống universal read.

Code tập trung trong single file `/mm/readahead.c`, nhưng vẫn quá lớn tôi không tiện paste ra, nên chỉ show phần core:

```c
/*
 * A minimal readahead algorithm for trivial sequential/random reads.
 */
static unsigned long
ondemand_readahead(struct address_space *mapping,
                   struct file_ra_state *ra, struct file *filp,
                   bool hit_readahead_marker, pgoff_t offset,
                   unsigned long req_size)
{
        struct backing_dev_info *bdi = inode_to_bdi(mapping->host);
        unsigned long max_pages = ra->ra_pages;
        unsigned long add_pages;
        pgoff_t prev_offset;

        /*
         * If the request exceeds the readahead window, allow the read to
         * be up to the optimal hardware IO size
         */
        if (req_size > max_pages && bdi->io_pages > max_pages)
                max_pages = min(req_size, bdi->io_pages);

        /*
         * start of file
         */
        if (!offset)
                goto initial_readahead;

        /*
         * It's the expected callback offset, assume sequential access.
         * Ramp up sizes, and push forward the readahead window.
         */
        if ((offset == (ra->start + ra->size - ra->async_size) ||
             offset == (ra->start + ra->size))) {
                ra->start += ra->size;
                ra->size = get_next_ra_size(ra, max_pages);
                ra->async_size = ra->size; // ##flag #0
                goto readit;
        }

        /*
         * Hit a marked page without valid readahead state.
         * E.g. interleaved reads.
         * Query the pagecache for async_size, which normally equals to
         * readahead size. Ramp it up and use it as the new readahead size.
         */
        if (hit_readahead_marker) {
                pgoff_t start;

                rcu_read_lock();
                start = page_cache_next_hole(mapping, offset + 1, max_pages);
                rcu_read_unlock();

                if (!start || start - offset > max_pages)
                        return 0;

                ra->start = start;
                ra->size = start - offset;	/* old async_size */
                ra->size += req_size;
                ra->size = get_next_ra_size(ra, max_pages);
                ra->async_size = ra->size; // ##flag #0
                goto readit;
        }

        /*
         * oversize read
         */
        if (req_size > max_pages)
                goto initial_readahead;

        /*
         * sequential cache miss
         * trivial case: (offset - prev_offset) == 1
         * unaligned reads: (offset - prev_offset) == 0
         */
        prev_offset = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;
        if (offset - prev_offset <= 1UL)
                goto initial_readahead; // ##flag #3

        /*
         * Query the page cache and look for the traces(cached history pages)
         * that a sequential stream would leave behind.
         */
        if (try_context_readahead(mapping, ra, offset, req_size, max_pages))
                goto readit;

        /*
         * standalone, small random read
         * Read as is, and do not pollute the readahead state.
         */
        return __do_page_cache_readahead(mapping, filp, offset, req_size, 0);

initial_readahead:
        ra->start = offset;
        ra->size = get_init_ra_size(req_size, max_pages);
        ra->async_size = ra->size > req_size ? ra->size - req_size : ra->size; // ##flag #1

readit:
        /*
         * Will this read hit the readahead marker made by itself?
         * If so, trigger the readahead marker hit now, and merge
         * the resulted next readahead window into the current one.
         * Take care of maximum IO pages as above.
         */
        if (offset == ra->start && ra->size == ra->async_size) {
                add_pages = get_next_ra_size(ra, max_pages);
                if (ra->size + add_pages <= max_pages) {
                        ra->async_size = add_pages;
                        ra->size += add_pages;
                } else {
                        ra->size = max_pages;
                        ra->async_size = max_pages >> 1;
                }
        }

        return ra_submit(ra, mapping, filp);
}
```

## 7. Chi tiết implementation

Mặc dù framework tổng thể nhìn đơn giản (nhờ thiết kế tinh tế của tác giả thuật toán), nhưng vẫn có nhiều chi tiết đáng suy ngẫm:

1. Làm thế nào để heuristic xác nhận các hành vi sequential read, interleaved read, random read trong framework?
2. Unnecessary readahead và readahead thrashing đã đề cập trong phần complexity được xử lý như thế nào?
3. Readahead window nên update như thế nào, giá trị lấy sao?

Trong đó vấn đề 3 sẽ trực tiếp ảnh hưởng đến chiến lược giải quyết vấn đề 1, 2.

### 7.1. Window khởi tạo và iteration

Trước tiên xem vấn đề 3, chiến lược update window. Hai function tương ứng `get_init_ra_size` / `get_next_ra_size`, chúng quyết định kích thước `ra->size`:

```c
/*
 * Set the initial window size, round to next power of 2 and square
 * for small size, x 4 for medium, and x 2 for large
 * for 128k (32 page) max ra
 * 1-8 page = 32k initial, > 8 page = 128k initial
 */
static unsigned long get_init_ra_size(unsigned long size, unsigned long max)
{
        unsigned long newsize = roundup_pow_of_two(size);

        if (newsize <= max / 32)
                newsize = newsize * 4;
        else if (newsize <= max / 4)
                newsize = newsize * 2;
        else
                newsize = max;

        return newsize;
}

/*
 *  Get the previous window size, ramp it up, and
 *  return it as the new window size.
 */
static unsigned long get_next_ra_size(struct file_ra_state *ra,
                                                unsigned long max)
{
        unsigned long cur = ra->size;
        unsigned long newsize;

        if (cur < max / 16)
                newsize = 4 * cur;
        else
                newsize = 2 * cur;

        return min(newsize, max);
}
```

- Đối với `size`: Tổng thể áp dụng chiến lược exponential (2x hoặc 4x), nếu giá trị hiện tại càng lớn thì càng conservative, và kích thước không vượt quá `max` (kích thước IO tối đa cho phép của filesystem hoặc device)
- Đối với `async_size`: Thường greedy cố gắng khả năng async tối đa, tức `async_size = size`, xem `##flag #0` và `##flag #1` trong flow `ondemand_readahead`

Mục đích thiết kế là đảm bảo chương trình dừng sequential access bất cứ lúc nào cũng có thể đạt được hit rate đáng kể.

### 7.2. Cache hit trong preread

Readahead cache hit là hành vi cache hit không cần thiết, vấn đề của nó sẽ giảm throughput của IO.

```c
// ra_submit -> __do_page_cache_readahead

/*
 * __do_page_cache_readahead() actually reads a chunk of disk.  It allocates
 * the pages first, then submits them for I/O. This avoids the very bad
 * behaviour which would occur if page allocations are causing VM writeback.
 * We really don't want to intermingle reads and writes like that.
 *
 * Returns the number of pages requested, or the maximum amount of I/O allowed.
 */
unsigned int __do_page_cache_readahead(struct address_space *mapping,
                struct file *filp, pgoff_t offset, unsigned long nr_to_read,
                unsigned long lookahead_size)
{
        struct inode *inode = mapping->host;
        struct page *page;
        unsigned long end_index;	/* The last page we want to read */
        LIST_HEAD(page_pool);
        int page_idx;
        unsigned int nr_pages = 0;
        loff_t isize = i_size_read(inode);
        gfp_t gfp_mask = readahead_gfp_mask(mapping);

        if (isize == 0)
                goto out;

        end_index = ((isize - 1) >> PAGE_SHIFT);

        /*
         * Preallocate as many pages as we will need.
         */
        for (page_idx = 0; page_idx < nr_to_read; page_idx++) {
                pgoff_t page_offset = offset + page_idx;

                if (page_offset > end_index)
                        break;

                rcu_read_lock();
                page = radix_tree_lookup(&mapping->i_pages, page_offset);
                rcu_read_unlock();
                if (page && !radix_tree_exceptional_entry(page)) {
                        /*
                         * Page already present?  Kick off the current batch of
                         * contiguous pages before continuing with the next
                         * batch.
                         */
                        if (nr_pages)
                                read_pages(mapping, filp, &page_pool, nr_pages, // ⭐
                                                gfp_mask);
                        nr_pages = 0;
                        continue;
                }

                page = __page_cache_alloc(gfp_mask);
                if (!page)
                        break;
                page->index = page_offset;
                list_add(&page->lru, &page_pool);
                if (page_idx == nr_to_read - lookahead_size)
                        SetPageReadahead(page); // ##flag #2
                nr_pages++;
        }

        /*
         * Now start the IO.  We ignore I/O errors - if the page is not
         * uptodate then the caller will launch readpage again, and
         * will then handle the error.
         */
        if (nr_pages)
                read_pages(mapping, filp, &page_pool, nr_pages, gfp_mask);
        BUG_ON(!list_empty(&page_pool));
out:
        return nr_pages;
}
```

Trong quy trình submit IO, `read_pages` (tức `mapping->a_ops->readpages`) cần các page không cache liên tục. Nếu trong phạm vi `[page_idx, page_idx + nr_to_read)` tồn tại page cache, thì cần thực hiện nhiều lần `readpages` tách rời.

Nếu cần giải quyết, phải quan sát quy luật của cached page: đa số cached page liên tục kề nhau. Do đó cần detect ra cache region đó và cấm khởi động readahead behavior bên trong.

Chiến lược giải quyết readahead cache hit của on-demand algorithm:

1. **Hạn chế điều kiện set flag**: Chỉ đánh dấu `PG_readahead` cho new readahead page, xem `##flag #2`
2. **Tránh thực hiện readahead routine thường xuyên**: Chỉ khi page miss hoặc trigger `PG_readahead` mới thực hiện readahead

*(Lưu ý: Đối với điểm 1, để tránh hành vi từ chối đánh dấu sai do chỉ có ít cache tồn tại, on-demand algorithm có thể thực hiện nghiêm ngặt hơn - chỉ khi tất cả page đều trigger readahead cache hit mới từ chối set `PG_readahead`, nhưng algorithm không sử dụng chiến lược nghiêm ngặt này)*

### 7.3. Readahead thrashing

Readahead thrashing xuất phát từ conflict với cơ chế page replacement/reclaim. On-demand readahead algorithm cho rằng readahead thrashing có xác suất cao xảy ra trong large-scale concurrent sequential access load.

Do đó cơ chế detect của nó phán đoán như sau:
- Đang sequential access
- Page access không có trong cache

Sau khi detect xác nhận, cơ chế bảo vệ readahead thrashing:
- Restart readahead tại vị trí hiện tại, thay vì single page small IO
- Dựa theo current read request, reset exponential window size

Implementation của nó merge với non-aligned behavior trong initial window flow, xem `##flag #3`.

### 7.4. Sequential read phức tạp hơn

Trong trường hợp ứng dụng thông thường, sequential read cũng không phải là tăng trưởng tuyến tính đơn giản:

- Trong **non-blocking IO**, hành vi có thể gọi là retry read, trong một số read request interval \\([offset_{start}, offset_{end})\\), \\(offset_{end}\\) luôn cố định, còn \\(offset_{start}\\) có thể không đổi hoặc tăng ít, như \\([0, 1000), [2, 1000), [6, 1000) \dots\\)

- Trong **concurrent processing flow**, IO của cùng một file có thể được xử lý bởi nhiều concurrent stream, hành vi này gọi là interleaved read. Một số read request interval sau khi merge thực sự là sequential read, nhưng từ góc độ filesystem có thể là \\([0, 2), [100, 103), [2, 5), [103, 105) \dots\\)

Chiến lược giải quyết của on-demand readahead algorithm rất đơn giản:

- Điều kiện trigger readahead theo page (actual access page) chứ không phải theo request
- Đối với retry read, vẫn có thể xử lý như linear sequential read  
- Đối với interleaved read, hạn chế điều kiện trigger readahead sẽ tránh bị coi là random read và bị force interrupt. Như do nhầm trigger `PG_readahead` mà vào readahead routine, thì kiểm tra phía trước một page cache có tồn tại không, nếu có thì chứng minh đây thực sự là stream liên tục về không gian (về thời gian là interleaved behavior), không xử lý readahead trùng lặp, không tồn tại thì readahead lại

*(Lưu ý: Cách xử lý interleaved read thực chất là tìm phương án bù đắp cho phương pháp đánh dấu `PG_readahead`)*

## 8. Tổng kết

Bài viết này đại khái đã sắp xếp core flow của thuật toán prefetch, cũng như các giải pháp thiết kế cho corner case khác nhau.

Vấn đề cốt lõi của readahead algorithm là: **Làm thế nào thiết kế một bộ thuật toán đơn giản, hiệu quả và có thể mở rộng cho các trường hợp phức tạp**.

Tính đơn giản và khả năng mở rộng nằm ở việc toàn bộ algorithm flow được thiết kế theo dạng pipeline, readability cực cao (bạn có thể trực tiếp thấy priority của bất kỳ cách xử lý nào). Nếu cần customize giải pháp phù hợp hơn một cách có mục tiêu, chỉ cần insert thêm algorithm vào pipeline.

Một khía cạnh khác là tối ưu về mặt thuật toán:

- **Độ phức tạp không gian**: Chỉ \\(O(1)\\), ít biến duy trì cũng có nghĩa là state transition rõ ràng hơn, tham số pipeline thực chất là `async_size`
- **Độ phức tạp thời gian**: Là passive processing càng nhiều càng tốt (thú vị là tuy gọi là readahead nhưng không bao giờ tích cực), cũng thông qua cách đánh dấu đơn giản giảm khả năng gọi readahead routine, thậm chí khéo léo tránh được state transition sai. Tôi nghĩ về ý nghĩa engineering cũng có giá trị tham khảo lớn.

## 9. Tài liệu tham khảo

Readahead algorithm vẫn còn không ít chi tiết nhỏ có thể khai thác, ví dụ xử lý oversized read, hoặc tìm ra potential sequential read dựa trên historical cache context, v.v. Quan tâm có thể xem paper của tác giả thuật toán Ngô Phong Quang [_Linux 内核中的预取算法_](https://cdmd.cnki.com.cn/Article/CDMD-10358-2009110818.htm), cũng như một số commit message liên quan, random post một vài link để tham khảo:

- [readahead: on-demand readahead logic](https://github.com/torvalds/linux/commit/122a21d11cbfda6d1e33cbc8ae9e4c4ee2f1886e)
- [readahead: introduce context readahead algorithm](https://github.com/torvalds/linux/commit/10be0b372cac50e2e7a477852f98bf069a97a3fa)  
- [readahead: make context readahead more conservative](https://github.com/torvalds/linux/commit/2cad401801978b16ac6e43f10b8d60039670fcbc)



