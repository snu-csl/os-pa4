# 4190.307 Operating Systems (Fall 2024)
# Project #4: Xswap: Compressed Swap for xv6
### Due: 11:59 PM, November 24 (Sunday)

## Introduction

The `xv6` kernel currently lacks support for swapping. This project introduces Xswap, a compressed memory cache for swap pages inspired by Linux's Zswap feature. By storing swap pages in a compressed format within RAM, Xswap reduces disk I/O and improves overall performance under memory pressure. The goal of this project is to understand the virtual memory subsystem in `xv6` and explore the challenges and design considerations involved in adding swapping functionality.

## Background

### Linux Zswap

Linux's Zswap is a compressed, in-memory cache for swap pages that improves system performance and efficiency under memory pressure. Instead of immediately writing pages to a slower swap device, Zswap compresses and stores them in RAM, reducing the need for costly disk I/O. This approach allows more data to remain in memory, leveraging the faster access speeds of RAM and deferring or even avoiding swap writes to disk. For more details on Zswap, please refer to [this page](https://docs.kernel.org/admin-guide/mm/zswap.html).

### RISC-V paging hardware

The RISC-V processor uses standard paging with a 4KB page size. Specifically, `xv6` runs on Sv39 RISC-V architecture, which uses 39-bit virtual addresses with a three-level page table structure. When paging is turned on in the supervisor mode, the `satp` register points to the physical address of the root page table. A page fault occurs when a process attempts to access a page with either an invalid page table entry (PTE) or invalid access permissions. 

When a trap occurs and control is transferred to the supervisor mode, the `scause` register is set with a code that indicates the event that triggered the trap, as shown in the following table.

|  Exception code | Description                     |
|:---------------:|:--------------------------------| 
| 0               | Instruction address misaligned  |
| 1               | Instruction access fault        |
| 2               | Illegal instruction             |
| 3               | Breakpoint                      |
| 4               | _Reserved_                      |
| 5               | Load access fault               |
| 6               | AMO address misaligned          |
| 7               | Store/AMO access fault          |
| 8               | Environment call (syscall)      |
| 9 - 11          | _Reserved_                      |
| 12              | __Instruction page fault__      |
| 13              | __Load page fault__             |
| 14              | _Reserved_                      |
| 15              | __Store page fault__        |
| >= 16           | _Reserved_                      |

Please focus on the following three events in the table above: Instruction page fault (12), Load page fault (13), and Store page fault (15). These events indicate page faults caused by instruction fetches, load instructions, or store instructions, respectively. On a page fault, RISC-V also provides the _virtual_ address that caused the fault through the `stval` register. Currently, no page faults occur in `xv6` because all the necessary code and data reside in physical memory. However, in this project, you will need to handle these page faults. Note that the values of the `scause` and `stval` registers can be read in  `xv6` by calling the `r_scause()` and `r_stval()` functions, respectively. 

In Sv39, each page table entry (PTE) is 8 bytes long and follows the format shown below.
```
63         54 53                          10  9  8  7   6   5   4   3   2   1   0            
+------------+------------------------------+-----+---+---+---+---+---+---+---+---+
|  Reserved  |  Physical Page Number (PPN)  | RSW | D | A | G | U | X | W | R | V |
+------------+------------------------------+-----+---+---+---+---+---+---+---+---+
```
The RSW field (bits 9-8) is reserved for use by supervisor software. The last 8 bits (bit 7-0) have the following meanings.
```
D (bit 7): Dirty bit. Set to 1 if the page has been written.
A (bit 6): Access bit. Set to 1 if the page has been read, written, or fetched.
G (bit 5): Global bit. Set to 1 if the mapping exists in all address spaces.
U (bit 4): User bit. Set to 1 if the page is accessible to user mode.
X (bit 3): Execution bit. Set to 1 if the page is executable.
W (bit 2): Write bit. Set to 1 if the page is writable.
R (bit 1): Read bit. Set to 1 if the page is readable.
V (bit 0): Valid bit. Set to 1 if the entry is valid.
```

### Xswap

Similar to Linux's Zswap, Xswap provides a compressed memory cache for swap pages. To enable this functionality, `xv6` now reserves a portion of physical memory for storing compressed swap-out pages. The following figure illustrates the modified physical memory map when using Xswap.

* Original `xv6` Physical Memory Map

  Physical memory in `xv6` begins at the physical address `0x80000000` and extends up to `PHYSTOP`, which marks the end of physical memory. The variable `end` points to the physical address of the 
  first available memory location. 

  ```
       PHYSTOP -> +---------------+ PHYSTOP: end of physical memory
                  |               |
                  |               |
                  |               |
                  |               |
                  |               |
           end -> +---------------+  end of kernel image
                  |  kernel data  |
                  +---------------+
                  |  kernel code  |
    0x80000000 -> +---------------+  start of physical memory
  ```

* New Physical Memory Map for Xswap
   
  ```
      ZMEMSTOP -> +---------------+  end of ZONE_ZMEM
                  |               |
                  |   ZONE_ZMEM   |  (space for compressed swap)
                  |               |
       PHYSTOP -> +---------------+  stat of ZONE_ZMEM
                  |  ZONE_NORMAL  |  (space for user pages)
                  |               |
  NORMAL_START -> +---------------+  NORMAL_START: start of ZONE_NORMAL
                  |  ZONE_FIXED   |  (space for kernel pages)
                  |               |
           end -> +---------------+  end of kernel image
                  |  kernel data  |
                  +---------------+
                  |  kernel code  |
    0x80000000 -> +---------------+  start of physical memory
  ```

  As you can see above, physical memory consists of the following zones for Xswap:
  
  | Zone                   | Usage                                                                                |
  |:-----------------------|:-------------------------------------------------------------------------------------|
  |`0x80000000`~`end`      | kernel code and data                                                                 |
  |`end`~`NORMAL_START`    | **`ZONE_FIXED`**: This area is reserved for unmovable pages used by the kernel. Any frames allocated within this zone are excluded from swapping. Examples include pages for kernel stacks, page tables, trapframes, pipe/device buffers, etc. |
  |`NORMAL_START`~`PHYSTOP`| **`ZONE_NORMAL`**: This zone is allocated for swappable pages used by user space processes. This includes user code/data/stack/heap pages. When this zone reaches its capacity, pages are swapped out to `ZONE_ZMEM`.           |
  |`PHYSTOP`~`ZMEMSTOP`    | **`ZONE_ZMEM`**: This zone serves as the compressed swap cache for swapped-out pages, storing them in compressed form to optimize memory usage. |     

### Managing `ZONE_ZMEM`

The physical memory in `ZONE_ZMEM` can be allocated either 2KB or 4KB units. When a victim page is selected for swap-out, it is first compressed. If the compressed size does not exceed 2KB, it is stored in a 2KB section of `ZONE_ZMEM`. Otherwise, the original content is copied into a 4KB section of `ZONE_ZMEM`. We use the [LZO data compression](https://github.com/nemequ/lzo) algorithm, which is also used in Linux's Zswap. The following LZO compression/decompression routines are available in the skeleton code (@ `kernel/lzo.c`), which are adapted from the LZO library used in Linux and tailored for `xv6` running on 64-bit little-endian RISC-V CPUs.

__SYNOPSYS__
```
  int lzo1x_compress(const unsigned char *src, uint32 src_len, unsigned char *dst, uint32 *dst_len, void *wrkmem);
```

__DESCRIPTION__

The `lzo1x_compress()` function compresses the memory block at `src` with the uncompressed length `src_len` to the address given by `dst` according to the LZO1X algorithm. The length of the compressed block is returned in the variable pointed by `dst_len`. The algorithm requires a separate working memory of size `LZO1X_1_MEM_COMPRESS` (16KB) internally. The last argument, `wrkmem`, specifies the starting address of this working memory. While it is technically possible for LZO compression to produce a result larger than the original data, our tests, conducted with `usertests` and others, showed that this did not occur. Therefore, it is safe to assume that the compression result will not be larger than the original data. 

__RETURN VALUE__

* This function always returns `LZO_E_OK`.
  
__SYNOPSYS__
```
  int lzo1x_decompress(const unsigned char *src, uint32 src_len, unsigned char *dst, uint32 *dst_len);
```

__DESCRIPTION__

The `lzo1x_decompress()` function decompresses the memory block at `src` with the compressed length `src_len` to the address given by `dst` according to the LZO1X algorithm. The length of the decompressed block is returned in the variable pointed by `dst_len`. On error, the number of bytes that have been decompressed so far will be returned. It is expected that the number of bytes available in the `dst` block is passed via the variable pointed by `dst_len`. 

__RETURN VALUE__

* `LZO_E_OK`: Success
* `LZO_E_INPUT_NOT_CONSUMED`: The end of the compressed block has been detected before all bytes in the compressed block have been used.
* `LZO_E_INPUT_OVERRUN`: The decompressor requested more bytes from the compressed block than available. Your data is corrupted.
* `LZO_E_OUTPUT_OVERRUN`: The decompressor requested to write more bytes to the uncompressed block than available. Either your data is corrupted, or you should increase the number of available bytes passed in the variable pointed by `dst_len`.
* `LZO_E_LOOKBEHIND_OVERRUN`: Your data is corrupted.
* `LZO_E_EOF_NOT_FOUND`: No EOF code was found in the compressed block. Your data is corrupted, or `src_len` is too small.
* `LZO_E_ERROR`: Any other error (data corrupted).

## Problem Specification

### Part 1. Implementing New Physical Memory Allocators (20 points)

Your first task is to implement the necessary memory allocators for each memory zone. 
For `ZONE_FIXED` and `ZONE_NORMAL`, all requests are 4KB in size, allowing us to reuse the existing physical memory allocators, `kalloc()` and `kfree()`, with  the target memory zone specified as shown below.

__SYNOPSYS__
```
  void *kalloc(int zone);
```

__DESCRIPTION__

Allocates a 4KB frame from the specified memory zone. The `zone` parameter can be either `ZONE_FIXED` or `ZONE_NORMAL`.

__RETURN VALUE__

* On success, returns the start address of the allocated page frame, aligned to the 4KB page boundary.
* If no page frame is available, returns 0. 

__SYNOPSYS__
```
  void kfree(void *pa, int zone);
```

__DESCRIPTION__

Frees the 4KB page frame starting at the specified address `pa` from the given memory zone. The `zone` parameter can be either `ZONE_FIXED` or `ZONE_NORMAL`. If the address `pa` does not belong to the specified zone or was not previously allocated by `kalloc()`, it should trigger `panic("kfree")`.

__RETURN VALUE__

* This function does not return any value. 

Unlike other memory zones, `ZONE_ZMEM` should support the allocation and release of 2KB memory blocks. To enable this, you need to implement the following new allocator for `ZONE_ZMEM`.

__SYNOPSYS__
```
  void *zalloc(int type);
```

__DESCRIPTION__

Allocates a physical memory block from `ZONE_ZMEM`. If `type` is `ZFULL`, a 4KB frame is allocated; if it is `ZHALF`, a 2KB frame is allocated. The allocator should maximize the number of available 2KB or 4KB frames. For example, allocating two 2KB frames should reduce the number of allocatable 4KB frames by at most one. Similarly, freeing two consecutive 2KB frames that fit into a 4KB frame should make an additional 4KB frame available.

__RETURN VALUE__

* On success, returns the starting address of the allocated frame. If `type` is `ZFULL`, the address should be aligned to a 4KB boundary; if `type` is `ZHALF`, to a 2KB boundary.
* If no page frame is available, returns 0. 

__SYNOPSYS__
```
  void zfree(void *pa, int type);
```

__DESCRIPTION__

Frees the allocated page frame starting at the specified address `pa`. If `type` is `ZFULL`, it indicates a 4KB page frame; if `type` is `ZHALF`, a 2KB page frame. If the address `pa` does not belong to `ZONE_ZMEM` or was not previously allocated by `zalloc()`, it should trigger `panic("zfree")`.

__RETURN VALUE__

* This function does not return any value.

To inspect the allocation status of physical memory, the skeleton code includes three global variables in `kernel/xswap.c`: `nalloc4k`, `zalloc4k`, and `zalloc2k`. The `nalloc4k` variable tracks the number of allocated 4KB frames in `ZONE_NORMAL`, while `zalloc4k` and `zalloc2k` represent the number of allocated 4KB and 2KB frames in `ZONE_ZMEM`, respectively. You must ensure these variables accurately reflect the number of allocated frames. 
The current values of these variables are displayed when you press `ctrl-x` on `xv6`. 

Additionally, the skeleton code defines two variables, `nswapin` and `nswapout`, which count the number of swap-ins and swap-outs, respectively.
These counters will be used in Part 2 to monitor swap activity. 

We introduce a new system call named `memstat()` (located in `./kernel/xswap.c`) that returns the values of these variables, as shown below.
Please complete the implementation of the `memstat()` system call. For Part 1, `nswapin` and `nswapout` can be left as zero.

__SYNOPSYS__
```
  int memstat(int *n4k, int *z4k, int *z2k, int *swapin, int *swapout);
```

__DESCRIPTION__

The `memstat()` system call returns the values of `nalloc4k`, `zalloc4k`, `zalloc2k`, `nswapin`, and `nswapout` to the memory locations specified by
the parameters `n4k`, `z4k`, `z2k`, `swapin`, and `swapout`, respectively. If any of the parameter is 0, the corresponding variable's value is ignored. 

__RETURN VALUE__

* Always returns the total number of allocated page frames in `ZONE_NORMAL` and `ZONE_ZMEM`, calculated as `nalloc4k + zalloc4k + zalloc2k`.

The final task in Part 1 is to patch the current `xv6` kernel to use the modified `kalloc()` and `kfree()` functions. 
Currently, `kalloc()` is called without specifying a memory zone. You will need to modify the kernel so that frames used by the kernel (e.g., kernel stack pages, trapframe pages, page table pages, pipe or device buffers, etc.) are allocated using `kalloc(ZONE_FIXED)`, while frames used for user processes (e.g., code, data, heap, and stack pages) are allocated using `kalloc(ZONE_NORMAL)`. This change should not affect the kernel's behavior; ensure that the kernel functions correctly, as it did previously.

### Part 2. Enabling Swapping Functionality (50 points)

The goal of Part 2 is to enable swapping in `xv6` by using `ZONE_ZMEM` as the backing store for swap pages, initially without applying any compression. The following provides a brief overview of the swapping operation.

* The `kalloc(ZONE_NORMAL)` function internally calls `swapout()` when it detects that there are no available frames in `ZONE_NORMAL`. In this case, `kalloc(ZONE_NORMAL)` returns the address of the frame generated by `swapout()` to its caller. Therefore, when swapping is enabled, `kalloc(ZONE_NORMAL)` will return 0 only when `swapout()` fails &mdash; meaning no memory is available in `ZONE_ZMEM`.
  
* The `swapout()` function chooses a victim frame based on the FIFO replacement policy. Once a victim frame is identified, it allocates a 4KB frame in `ZONE_ZMEM`, and copies the content of the victim frame into this newly allocated frame. The corresponding page table entry (PTE) is then marked as invalid. You may utilize the reserved (RSW) field in the PTE to indicate that the corresponding page has been swapped out.
  
* If the swapped-out pages are later needed, the kernel calls the `swapin()` function, which transfers the contents of the frame stored in `ZONE_ZMEM` back into the frame in `ZONE_NORMAL`, updating the corresponding PTE as well. Note that `swapin()` may require an additional `swapout()` to make space for bringing the previously swapped-out page back into `ZONE_NORMAL`.
  
* User processes should not directly access frames in `ZONE_ZMEM`. If a frame in `ZONE_ZMEM` is required, it must first be brought back into `ZONE_ZMEM` via `swapin()`. Once the data is copied, the frame in `ZONE_ZMEM` is immediately released.

* You need to ensure that `nswapin` and `nswapout` accurately track the number of swap-in and swap-out operations performed, respectively.


### Part 3. Supporting Compressed Swapping and Ensuring No Memory Leaks (20 points)

In Part 3, you will extend the swapping functionality to support compressed swapping.
The goal is to enhance memory efficiency by compressing swap pages before storing them in `ZONE_ZMEM`.
Compressed data will be stored in a 2KB frame within `ZONE_ZMEM` if the compressed size is 2KB or less; otherwise, the original (uncompressed) data will be copied into a 4KB frame in `ZONE_ZMEM`, as is done in Part 2. 

To receive full credit for this project, you must also ensure that your implementation is free from memory leaks. 
Whenever you return to the shell after executing a command, the sum of `nalloc4k`, `zalloc4k`, and `zalloc2k` should remain exactly the same. Otherwise, it means either you forgot to free some page frames or you deallocated some page frames you shouldn't. 

Before submitting your code, please make sure your implementation does not have any memory leaks by monitoring the total number of allocated page frames after running the following programs:

```
$ forktest
$ cat README | wc
$ cat README | wc | wc | wc
$ usertests -q
$ ...
```

The skeleton code also includes two additional system calls, `ktest1()`, and `ktest2()`, in the `kernel/ktest.c` file. These system calls will be used for grading; please do not modify them. 

### Part 4. Design Document (10 points)

Prepare a design document detailing your implementation in a single PDF file. Your document should include the following sections:

1. New data structures
   * Describe any new data structures introduced.
   * Explain why these data structures were necessary and how they contribute to the implementation.
2. Algorithm design
   * Provide an overview of the overall flow of the `swapin()` and `swapout()` functions.
   * Describe any corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation, if any.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.

### **Bonus** (up to an additional 20 points)

If you ensure that swapping works correctly on a multi-core system (i.e., `CPUS` > 1 in `Makefile`), you can earn an additional 20-point bonus.
We will use various test cases, including `usertests`, to evaluate whether it functions correctly on a multi-core system. To qualify for this bonus, wrap any code changes with `#ifdef MULTI` and `#endif` and include the relevant details in your design document.
 
## Restrictions

* For Part 1 ~ 3, you may assume that `xv6` is running on a single-core system (i.e., `CPUS` = 1 in `Makefile`).
* For Part 1 and Part 2, we will run your code without any special compiler options. However, we will use the `-DPART3` compiler option to check whether your code successfully implements Part 3 of this project. Therefore, please enclose any code modifications for Part 3 within the `#ifdef PART3` and `#endif` macros.
* For the bonus, we will use the `-DMULTI` compiler option along with `-DPART3` to verify whether your code works correctly on a multi-core system. 
* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`
* You are required to modify only the files in the `./kernel` directory. If you have created your own test cases, place them in the `./user` directory and mention them in your documentation. Any other changes will be ignored during grading.

## Skeleton Code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa4
```
After downloading, you must first set your `STUDENTID` in the `Makefile` again.

The skeleton code includes new files, `lzo.c`, `xswap.c`, and `xswap.h`, located in the `./kernel` directory. 
The `lzo.c` file contains the LZO compression/decompression library, while `xswap.h` and `xswap.c` have new macros, variables, and necessary system calls related to swapping. You are free to add any additional code to `xswap.h` and `xswap.c` files.

The `pa4` branch includes a user-level program called `swaptest`, with its source code located in `./user/swaptest.c`. 
The `swaptest` program performs a stress test to verify the functionality of swap-in and swap-out operations by repeatedly accessing an array `a` that spans 64 pages. In each iteration, it sequentially accesses blocks within each page across all 64 pages, ensuring frequent swapping in and out. After a specified number of iterations, `swaptest` verifies consistency by checking if each element in the array has been correctly incremented. 

## Examples

The skeleton code initializes the start address of `ZONE_NORMAL` at physical address `0x81000000`, as defined by `NORMAL_START` in `kernel/memlayout.h`. The sizes of `ZONE_NORMAL` and `ZONE_ZMEM` are specified by the `MEM` and `ZMEM` macros in the `Makefile`, with default values of 64 pages and 32MB, respectively. 

The following shows the output when you run the `swaptest` program on the skeleton code. 
Upon boot, the skeleton code displays information about the physical memory layout. When you press `ctrl-x`, `xv6` prints the current values of `total`, `nalloc4k`, `zalloc4k`, `zalloc2k`, `swapin` and `swapout`. 
The `swaptest` program itself also calls the `memstat()` system call at the start and end of its execution. 
Currently, the `swaptest` program appears to be running without any issues. However, at the moment, `swaptest` is using physical memory from `ZONE_FIXED`. You need to modify this so that it uses `ZONE_NORMAL`, and when memory in `ZONE_NORMAL` is insufficient, it should utilize `ZONE_ZMEM` as swap space. 

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

Physical memory layout:
Kernel:      0x80000000 - 0x80022000 (0 MB, 34 pages)
ZONE_FIXED:  0x80022000 - 0x81000000 (15 MB, 4062 pages)
ZONE_NORMAL: 0x81000000 - 0x81040000 (0 MB, 64 pages)
ZONE_ZMEM:   0x81040000 - 0x83040000 (32 MB, 8192 pages)
init: starting sh
$ total: 0, nalloc4k: 0, zalloc4k: 0, zalloc2k: 0, swapin: 0, swapout: 0   <-- ctrl-x
$ swaptest
Allocated frames (start): 0
k = 0
k = 1
k = 2
k = 3
k = 4
k = 5
k = 6
k = 7
k = 8
k = 9
swaptest: OK
Allocated frames (end): 0
$ total: 0, nalloc4k: 0, zalloc4k: 0, zalloc2k: 0, swapin: 0, swapout: 0   <-- ctrl-x
$ QEMU: Terminated                                                         <-- ctrl-a x
```

The following is an example output when you have successfully implemented all the requirements of this project. Note that
- the starting address of the `ZONE_FIXED` area may vary depending on the amount of static data allocated to the kernel.
- the exact values of `zalloc4k` and `zalloc2k` may vary, but their sum must be the same.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

Physical memory layout:
Kernel:      0x80000000 - 0x80048000 (0 MB, 72 pages)
ZONE_FIXED:  0x80048000 - 0x81000000 (15 MB, 4024 pages)
ZONE_NORMAL: 0x81000000 - 0x81040000 (0 MB, 64 pages)
ZONE_ZMEM:   0x81040000 - 0x83040000 (32 MB, 8192 pages)
init: starting sh
$ total: 9, nalloc4k: 9, zalloc4k: 0, zalloc2k: 0, swapin: 0, swapout: 0   <-- ctrl-x
$ swaptest
Allocated frames (start): 78
k = 0
k = 1
k = 2
k = 3
k = 4
k = 5
k = 6
k = 7
k = 8
k = 9
swaptest: OK
Allocated frames (end): 78
$ total: 9, nalloc4k: 4, zalloc4k: 0, zalloc2k: 5, swapin: 10482, swapout: 10513    <-- ctrl-x
$ QEMU: Terminated                                                                  <-- ctrl-a x  
```

## Tips

* Read Chap. 3 and 4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf) to understand the virtual memory subsystem and page-fault exceptions in `xv6`.
  
* Use the following programs to test your implementation. Note that `usertests` has been slightly modified to utilize the `memstat()` system call to check for memory leaks. Additionally, some test cases that allocate until the physical memory is completely exhausted have been excluded from `usertests`. 
  - `$ forktest`
  - `$ swaptest`
  - `$ usertests -q`

* For your reference, the following roughly shows the amount of changes you need to make for this project assignment. Each `+` symbol indicates 1~10 lines of code that should be added, deleted, or altered.
   ```
   kernel/defs.h        |  ++
   kernel/syscall.c     |  +
   kernel/exec.c        |  ++
   kernel/file.c        |  +
   kernel/kalloc.c      |  ++++++++
   kernel/pipe.c        |  +
   kernel/proc.c        |  +
   kernel/riscv.h       |  +
   kernel/sysfile.c     |  +
   kernel/trap.c        |  +
   kernel/virtio_disk.c |  +
   kernel/vm.c          |  ++++++++++
   kernel/xswap.c       |  ++++++++++++++++++++++++++++++
   kernel/xswap.h       |  +++++++
   ```

## Hand in instructions

* First, make sure you are on the `pa4` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file to the submission server. Additionally, your design document should be uploaded as the report for this project assignment.

* The total number of submissions for this project assignment will be limited to 50. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button.

* The `sys` server will no longer accept submissions after three days (72 hours) past the deadline.
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, it is likely due to an error in the automated process. The TAs will check if the script is running properly, but since that is a ___manual___ process, please do not expect it to be completed immediately.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use on the QnA board of the submission server before the next project assignment is announced. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)


