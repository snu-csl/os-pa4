# 4190.307 Operating Systems (Fall 2025)
# Project #4: P<sup>3</sup>: Per-Process Paging
### Due: 11:59 PM, November 30 (Sunday)

## Introduction

In this project, you will implement __per-process paging__ in `xv6`, enforcing a per-process limit on physical memory usage; when a process exceeds its quota, excess pages must be evicted to the swap device and brought back on demand upon page faults. The choice of page replacement policy is crucial because it dictates which pages are evicted, directly affecting page-fault rates, I/O overhead, and overall performance. The goal of this project is to understand `xv6`'s virtual memory subsystem and explore the challenges and design trade-offs involved in supporting swapping. 

## Background

### RISC-V paging hardware

The RISC-V processor uses standard paging with a 4 KiB page size. Specifically, `xv6` runs on Sv39 RISC-V architecture, which uses 39-bit virtual addresses with a three-level page table structure. When paging is turned on in the supervisor mode, the `satp` register points to the physical address of the root page table. A page fault occurs when a process attempts to access a page with either an invalid page table entry (PTE) or invalid access permissions. 

When a trap transfers control to the supervisor mode, the `scause` register is set with a code identifying the triggering event, as summarized in the table below.


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
| 15              | __Store page fault__            |
| >= 16           | _Reserved_                      |

Please focus on the following three events in the table: Instruction page fault (12), Load page fault (13), and Store page fault (15). These events indicate page faults caused by instruction fetches, load instructions, or store instructions, respectively. On a page fault, RISC-V also provides the _virtual_ address that caused the fault through the `stval` register. The values of the `scause` and `stval` registers can be read in `xv6` by calling the `r_scause()` and `r_stval()` functions, respectively. 
In the original `xv6`, page faults occur only for lazily allocated heap pages created via `sbrk(n, SBRK_LAZY)`, since all other regions (code, data, stack, and heap with `SBRK_EAGER`) are already kept resident in physical memory. 

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

### Swap Device

A swap device is a raw block device that serves as secondary backing store for virtual memory. When physical memory is scarce, selected pages are evicted from RAM to the swap device, and later faulted back on demand. The swap device uses a simple layout: block 0 holds a small metadata header (a _superblock_ with fields such as a magic number and device size), and the remaining blocks are partitioned into fixed-size __swap slots__—typically one slot per virtual page (e.g., 4 KiB)—as shown below:
```
   +----------+----------+----------+----------+----------+
   |swap      | swap     | swap     |          | swap     |
   |superblock| slot 0   | slot 1   |    ...   | slot n-1 |
   |(4KiB)    | (4KiB)   | (4KiB)   |          | (4KiB)   |
   +----------+----------+----------+----------+----------+
```

The kernel maintains a mapping from each swapped-out page to its slot, writes the page's contents into that slot on eviction, and reads them back on a page fault. 

## Problem Specification

### Part 1. `rss_set()/rss_get()/rss_stat()` system calls (20 points)

First, you are required to implement the `rss_set()`, `rss_get()`, and `rss_stat()` system calls. Their system call numbers are assigned to 40, 41, and 42, respectively, in `kernel/syscall.h`.

__NAME__

* `rss_set() ` -- set the resident set size (RSS) quota 
* `rss_get() ` -- get the current RSS quota
* `rss_stat()` -- retrieve paging statistics

__SYNOPSYS__
```C
    // In "user/user.h"
    struct rss_stats {
      int  quota;              // RSS quota
      int  rss;                // Resident set size 
      int  nrss;               // Non-resident (swapped) set size
      int  faults;             // Number of page faults
      int  swapins;            // Number of swap-ins
      int  swapouts;           // Number of swap-outs
    };

    int rss_set(int quota);
    int rss_get(void);
    int rss_stat(struct rss_stats *r);
```

__DESCRIPTION__

Resident Set Size (RSS) indicates the number of a process's pages that are currently resident in physical memory. Each process starts with a default RSS quota, `DEFAULT_RSS_QUOTA` (20 pages; see `kernel/proc.h`). These interfaces control and query per-process paging state, including the configured quota, the current RSS, and related paging statistics.

* `rss_set(quota)` sets the allowed RSS quota (in pages) for the calling process. The `quota` should be a positive integer. If the process's current RSS exceeds the new quota, the kernel should evict surplus pages to the swap device (according to the replacement policy) before returning.

* `rss_get()` returns the current RSS quota (in pages) for the calling process. 

* `rss_stat()` writes the calling process's paging statistics into the user-supplied `struct rss_stats` pointed to by `r`. Note that the kernel maintains these per-process counters in `p->rss` within the `proc` structure as shown below (cf. `kernel/proc.h`). The reported paging statistics include: the RSS quota (`p->rss.quota`), the current RSS (`p->rss.rss`), the non-resident set size (i.e., the number of swapped out pages) (`p->rss.nrss`), the number of page faults (`p->rss.faults`), the number of swap-ins (`p->rss.swpins`), and the number of swap-outs (`p->rss.swapouts`).
  
  ```C
  struct proc {
    struct spinlock lock;
    ...
  #ifdef SNU
    struct rss_stats rss;       // Per-process paging statistics
  #endif
  };
  ```

__RETURN VALUE__

* `rss_set()` returns 1 on success and 0 on invalid argument.
* `rss_get()` returns the current RSS quota (in pages).
* `rss_stat()` returns 1 on success and 0 if `r` is an invalid pointer.

For Part 1, grading focuses exclusively on correct RSS accounting (`p->rss.rss`) on a single-hart machine using the provided skeleton code. No swapping functionality is required for this part. Your implementation must track the current RSS (`p->rss.rss`) of each process precisely across relevant system calls such as `open()`, `exec()`, `sbrk()`, `exit()`, etc. Only user-space pages contribute to RSS; kernel memory (e.g., trapframe, trampoline, kernel stack, etc.) must not be included. 

### Part 2. Per-Process Paging with FIFO Replacement Policy (50 points)

The next task is to extend your implementation to enforce RSS quotas via swapping: whenever a process's resident set exceeds its quota, the kernel must evict some of that process's pages to the swap device and fault them back on demand. Only user-space pages in the process's virtual address range [0, `p->sz`) are eligible for eviction (including the user stack guard page). Pages outside this range (e.g., the trampoline and trapframe page) or pages with kernel-only mappings (e.g., the kernel stack and its guard page) are never evicted. Part 2 targets a single-hart machine (`CPUS := 1` in `Makefile`).

The skeleton code already attaches a second virtio-blk disk (default 32 MiB) to QEMU as the swap device. A small utility, `mkswap` (`mkfs/mkswap.c`), formats this device by writing a swap magic number and the number of available swap slots to its superblock. During boot, `xv6` calls `swapinit(SWAPDEV)` (right after the filesystem is initialized in `forkret() @ kernel/proc.c`) to validate the device. For data movement, the kernel can access swap slot _s_ directly using `swapread(pa, s)` or `swapwrite(pa, s)`, where `pa` is the physical address of a 4 KiB buffer (one page).

Implement a per-process FIFO replacement policy as the baseline: every process maintains its own FIFO queue of resident user page frames, and the oldest eligible resident page is selected as the victim and swapped out.
When a process's RSS exceeds its quota, the kernel selects a victim page from that process's FIFO queue, allocates a free swap slot, writes the page's contents to that slot, and updates the page's PTE to mark it non-resident. 
If the process later touches that virtual address, a page fault occurs; the page fault handler detects the swapped-out page, allocates a new page frame, reads the swap slot's contents back into memory, restores the corresponding PTE, and frees the swap slot (we assume immediate reclamation rather than leaving stale slots). Newly swapped-in pages are appended to the process's FIFO queue, while evicted pages are removed, keeping the queue consistent across map/unmap, eviction, and page fault paths. 
Throughout these steps, paging statistics (`p->quota`, `p->rss`, `p->nrss`, `p->faults`, `p->swapins`, and `p->swapouts`) must be maintained accurately. 

A process's RSS can exceed its quota while the kernel handles system calls such as `rss_set()`, `fork()`, `exec()`, `sbrk()`, etc., or when a swapped-out page is brought back on a page fault. These events may temporarily raise RSS above the quota during kernel processing; however, before returning to user space, the kernel must swap out enough pages to bring RSS back within the configured quota.

`xv6` already supports a limited form of demand paging via the `SBRK_LAZY` flag: the heap can grow by reserving address space now and allocating the physical page frame later on first touch, which is handled in `vmfault() @ kernel/vm.c`. Your swapping code must coexist with this behavior: the initial page fault on a lazily allocated heap address is still serviced by `vmfault()`, and must not be counted as swap-in. Once that page becomes resident, it may later be evicted due to RSS pressure. If a fault occurs on that swapped-out, lazily allocated heap page, it must be handled by your swap-in path (not `vmfault()`). 

To cleanly separate Part 2 from Part 1, the kernel will be compiled with the C flag `-DSWAP` when grading Part 2. Therefore, every code change specific to Part 2 must be wrapped with `#ifdef SWAP  ...  #endif`, as shown below. Please make sure your Part 1 build (without `-DSWAP`) must still compile and run unchanged. 
```C
#ifdef SWAP
  // Part 2 code here...
#endif
```

You will receive full credit (50 points) for Part 2 only if your implementation passes quick usertests (`usertests -q`) on a single-hart machine.


### Part 3. Multi-hart Support without Memory/Space Leaks (20 points)

Finally, make your implementation correct and deadlock-free on multi-hart systems. Also, your implementation must exhibit no memory leaks and no swap-space leaks. All page frames and kernel allocations must be freed on normal execution paths and on all error/rollback paths without double frees. Likewise, every allocated swap slot must be released immediately on successful swap-in and on normal or aborted process termination.

We internally use two global variables, `freemem` (for memory) and `freeswap` (for swap space), to detect leaks. 
 * `freemem`: Decremented exactly once on each successful `kalloc()` and incremented exactly once on each corresponding `kfree()`. This logic is already present in the skeleton code — do not change it.
 * `freeswap`: You must maintain this counter to reflect the number of free swap slots. Decrement exactly once when a swap slot is successfully allocated; increment exactly once when a swap slot is freed.
Whenever control returns to `sh` after running a program, both counters must match their values before the program started.
  
Full credit (20 points) requires passing quick usertests (`usertests -q`) on a multi-hart system (e.g., `CPUS` > 1 in `Makefile`) with no memory or swap-space leaks.

### BONUS: Bring Your Own Replacement Policy (Up to 20 points)

If your implementation meets all Part 3 requirements, you may earn up to 20 bonus points by designing and integrating your own page replacement policy that minimizes page faults on workloads exhibiting memory-access locality. Your policy should plug cleanly into your per-process paging implementation and remain correct, deadlock-free, and leak-free under the same testing conditions as Part 3.
You may track lightweight per-page metadata and/or maintain auxiliary state, provided overhead stays modest and the design is cleanly explained in your design document. 

For a diverse set of workloads, we will rank submissions by the average ratio of page faults (lower is better) across all workloads, specifically using the geometric mean of each workload’s page-fault ratio relative to the FIFO policy. The top 10 submissions will receive a 20% bonus, and the next 10 submissions will receive a 10% bonus.
To qualify for this bonus, wrap any code changes with `#ifdef BONUS  ... #endif` and document the policy's mechanics, rationale, and expected behavior in your design document.

### Part 4. Design Document (10 points)

Along with your code, submit a design document in a single PDF file. Your document should include the following sections.

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones.
   * Explain why these data structures/modifications were necessary and how they contribute to the implementation.
2. Algorithm design
   * Explain when you enforce the RSS quota.
   * Explain the overall flow of the swap-in and swap-out functions.
   * Describe how the FIFO replacement policy is implemented.
   * Explain how you manage swap space allocation and deallocation.
   * Explain your own replacement policy, if any, and the rationale behind it.
   * Summarize any additional metrics you collect, if any, and explain their roles.
   * Describe all corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.
   * Explain which part of the project consumed most of your time and why.
  
## Restrictions

* For Part 1 and 2, you may assume that `xv6` is running on a single-hart system (i.e., `CPUS := 1` in `Makefile`).
* For Part 1, we will run your code without any special compiler options. For Part 2 and 3, however, we will use the `-DSWAP` compiler flag. Therefore, please enclose any code modifications for Part 2 and 3 within the `#ifdef SWAP` and `#endif` macros.
* For the bonus, we will use the `-DBONUS` compiler flag along with `-DSWAP` to verify whether your code works correctly on a multi-hart system.
* On swap-in, the swap slot is freed immediately. Each allocated swap slot is owned by exactly one PTE and must never be referenced by more than one PTE. Similarly, each allocated user page frame is owned by exactly one PTE and must never be shared by more than one PTE. 
* Any attempt to artificially reduce swap-in/swap-out activity will incur penalties. Our goal is to make swap-in/swap-out counts be affected only by the page replacement policy. Examples include (but are not limited to):
  * Failing to reclaim the corresponding swap slot after a swap-in.
  * Servicing code-page faults by reading directly from the file instead of performing proper swap-out/swap-in.
  * Reading directly from a parent's swap slot into a child's page frame during `fork()`.
  * Simply discarding unmodified, zero-filled heap pages and reinitializing them later without performing the required swap-out/swap-in.
  * Intentionally tampering with paging statistics, etc.
* Do not change the allocation order, deallocation order, access order, or modification order of user pages inside every syscall in skeleton code.
* When handling a system call, if there are user pages that were swapped out, the kernel must not incrementally enforce the quota during the syscall. Instead, it must first read in all such pages, and enforce the quota only once after all required user page accesses have finished(right before returning from the syscall).
* exec() does not clear the statistics for faults, swap-ins, or swap-outs.
* The quota set by the rss_set() system call may vary, but it must not exceed 400.
* The line starting with "#define DEFAULT_RSS_QUOTA" must appear in proc.h. The value of DEFAULT_RSS_QUOTA may change, but it must not exceed 20.
* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`
* You are required to modify only the files listed below. Any other changes will be ignored during grading.
   ```
   kernel/defs.h        |  ++
   kernel/exec.c        |  ++
   kernel/fs.c          |  +
   kernel/pipe.c        |  +
   kernel/proc.h        |  +
   kernel/proc.c        |  ++++
   kernel/riscv.h       |  +
   kernel/swap.h        |  +
   kernel/swap.c        |  ++++++++++++++++++++
   kernel/trap.c        |  +
   kernel/vm.c          |  ++++
   ```
## Tips

* You can repurpose the RSW field (PTE bits 9-8) to store software state. For example, use one bit to mark that a page is swapped out.
* QEMU emulates the PTE's (Access) and D (Dirty) bits, so you can sample them to infer recent references and writes when designing your own replacement policy.
  
* Read Chap. 3 and 5 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2025-2/book-riscv-rev5.pdf) to understand the virtual memory subsystem and page-faults in `xv6`.

## Skeleton Code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa4
```
After downloading, you must first set your `STUDENTID` in the `Makefile` again.

### Setting Up the Swap Device

We first create a raw image file (default 32 MiB) that will be attached as the second disk in QEMU as follows:
```sh
$ qemu-img create -f raw swap.img 32M
``` 
The skeleton code adds `mkfs/mkswap.c`, a tiny host utility that initializes `swap.img` by writing a swap superblock into block 0 (4 KiB). The superblock contains a magic number (`0x4190307`, the course number!) and the total number of 4 KiB blocks in the device (including the superblock). The `xv6` kernel later validates this header during boot.

To attach the image as a secondary virtio-blk device, the `Makefile` adds the following QEMU options when invoking QEMU:
```Makefile
# SNU
QEMUOPTS += -drive file=swap.img,if=none,format=raw,id=x1
QEMUOPTS += -device virtio-blk-device,drive=x1,bus=virtio-mmio-bus.1
```

At boot, the kernel calls `swapinit(SWAPDEV)` immediately after the filesystem is initialized (in `forkret()`), which (1) reads and checks the superblock, (2) computes the number of available swap slots (one slot = 4 KiB), and (3) initializes in-kernel metadata.

In addition, the driver is extended to support multiple disks: `kernel/virtio_disk.c` now maintains a `vdisk[]` array with one entry per device (MMIO base, virtqueue state, descriptor rings, `vdisk_lock`, and per-request info).
The second device's MMIO region (e.g., `VIRTIO1`) is mapped in `kvmmake()` (`kernel/vm.c`), its interrupt (e.g., `VIRTIO1_IRQ`) is enabled in `plicinit()/plicinithart()` (`kernel/plic.c`), and recognized in `devintr()` (`kernel/trap.c`). 
At the block layer, buffers carry a device number, so `bread()/bwrite()` are routed to the correct virtio instances: `ROOTDEV` to disk 0 (filesystem) and `SWAPDEV` to disk 1 (swap). 

Finally, the skeleton code exposes two helper functions to access a swap slot by index:
```C
void swapread(uint64 pa, int slot);
void swapwrite(uint64 pa, int slot);
```

### Leak Detection with `freemem`/`freeswap`

We detect resource leaks using two kernel-level global counters: `freemem` (free physical page frames) and `freeswap` (free swap slots). The `freemem` counter is already maintained in the skeleton: it is decremented exactly once on each successful `kalloc()` and is incremented exactly once on the corresponding `kfree()`.
The `freeswap` counter is initialized when `xv6` initializes the swap device (in `swapinit()`), but it is your responsibility to keep it accurate thereafter: decrement it once when a swap slot is successfully allocated, and increment it once when a swap slot is freed.

For convenience, we slightly extended `procdump()` (in `kernel/proc.c`) so that pressing ctrl-p (`^p`) in QEMU prints both the per-process paging statistics and the global counters, as shown in the following example:
```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -drive file=swap.img,if=none,format=raw,id=x1 -device virtio-blk-device,drive=x1,bus=virtio-mmio-bus.1

xv6 kernel is booting

hart 1 starting
hart 2 starting
swap magic: 4190307
swap blocks: 8192
init: starting sh
$ ^p
1 sleep  init Q:20 R:0 N:0 F:0 I:0 O:0
2 sleep  sh Q:20 R:0 N:0 F:0 I:0 O:0
freemem:  32533
freeswap: 8191
```

Here, `Q`, `R`, `N`, `F`, `I`, and `O` denote `p->rss.quota`, `p->rss.rss`, `p->rss.nrss`, `p->rss.faults`, `p->rss.swapins`, and `p->rss.swapouts`, respectively. The absolute value of `freemem` may vary across builds (e.g., due to different amounts of static data), but for leak checking, the important property is that after a program finishes and control returns to `sh`, the before/after values of `freemem` and `freeswap` should be identical, indicating no memory or swap-space leaks.

### A simple test utility: `vmtest` 

In the skeleton code, there is a user-space program called `vmtest` (`user/vmtest.c`). It declares a 4 KiB-aligned 2-D integer array `a[N][1024]` so that each row occupies exactly one page (with `N=20`, the array spans 20 pages). 
The program prints the array's start/end address, then makes two sequential passes over the array: a write pass that stores to `a[i][i]` (touching one new page per iteration) and a read pass that loads the same locations to verify correctness. After each access, it calls `rss_stat()` to dump the current paging statistics, and at the end it checks that the accumulated sum matches the expected value, catching any swap-related data corruption.

Although the array spans 20 pages, the process already holds other user pages (text/data/stack), so the RSS will exceed the default quota, triggering evictions under the FIFO policy. Because the access pattern is strictly sequential, FIFO evicts the earliest pages; thus every second-pass read will incur at least one page fault and the associated swap-in. Also note that page faults may arise not only for pages in `a[]`, but also for other user mappings (code, data, and stack) that the kernel may have evicted under quota pressure. 

### The `meminfo()` system call and modifications to `usertests`

We will primarily use quick usertests (`usertests -q`) to check the correctness of your implementation. The suite has been slightly modified in two ways: (1) test cases that are not compatible with per-page swapping—especially those that deliberately exhaust all physical RAM—have been removed; and (2) some less-relevant test cases were dropped to reduce overall test time.

The original `usertests` relied on `countfree()`, which repeatedly called `sbrk()` until failure to estimate the amount of free memory. Instead, we added a new system call, `meminfo()` (syscall number 30, see `sys_meminfo()` in `kernel/swap.c`), which atomically returns the sum of `freemem` and `freeswap`. `usertests` snapshots this value before each subtest and after it completes; equality indicates no memory or swap-space leaks for that subtest.

## Hand in instructions

* First, make sure you are on the `pa4` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. In addition, you must also upload your design document as a PDF file for this project assignment.

* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after some time, it is likely due to an error in the automated process. The TAs will verify whether the script is running correctly, but since this check must be performed __manually__, please understand that it may not be completed immediately.

       
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


