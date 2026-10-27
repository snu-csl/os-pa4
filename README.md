# 4190.307 Operating Systems (Fall 2026)
# Project #4: `mmap()` with Huge Pages
### Due: 11:59 PM, November 19 (Thursday)

## Introduction

In this project, you will implement a simplified version of the `mmap()` system call using 2 MiB huge pages supported by the RISC-V architecture. The goal of this project is to understand the paging hardware of the RISC-V processor and how the operating system manages virtual-to-physical address mapping to improve memory efficiency.

## Background

### `mmap()` system call

In UNIX-based systems, the `mmap()` system call creates a new mapping in the virtual address space of the calling process. In *file-backed mapping*, `mmap()` maps a region of the process's virtual address space to a file, allowing file I/O to be treated as memory access. On the other hand, in *anonymous mapping*, `mmap()` maps a region of the process's virtual memory space that is not backed by any file, with the contents being initialized to zero. In this project, we focus on a version of the `mmap()` system call that supports anonymous mapping only. For additional details on `mmap()`, please refer to its manual page using the `$ man 2 mmap` command. 

### RISC-V paging hardware

The RISC-V processor uses standard paging with a 4 KiB base page size. Specifically, `xv6` runs on the Sv39 RISC-V architecture, which uses 39-bit virtual addresses with a three-level page table structure. When paging is turned on in supervisor mode, the `satp` register points to the physical address of the root page table. A page fault occurs when a process attempts to access a page with either an invalid page table entry (PTE) or insufficient access permissions. 

When a trap transfers control to supervisor mode, the `scause` register is set with a code that identifies the triggering event, as summarized in the table below.


|  Exception code | Description                     |
|:---------------:|:--------------------------------| 
| 0               | Instruction address misaligned  |
| 1               | Instruction access fault        |
| 2               | Illegal instruction             |
| 3               | Breakpoint                      |
| 4               | Load address misaligned         |
| 5               | Load access fault               |
| 6               | Store/AMO address misaligned    |
| 7               | Store/AMO access fault          |
| 8               | Ecall from U-mode or VU-mode    |
| 9               | Ecall from HS-mode              |
| 10              | Ecall from VS-mode              |
| 11              | Ecall from M-mode               |
| 12              | __Instruction page fault__      |
| 13              | __Load page fault__             |
| 14              | _Reserved_                      |
| 15              | __Store/AMO page fault__        |


Please focus on the following two events in the above table: Load page fault (13) and Store/AMO page fault (15). These events indicate page faults caused by load instructions or store instructions, respectively. On a page fault, RISC-V also provides the _virtual_ address that caused the fault in the `stval` register. The values of the `scause` and `stval` registers can be read in `xv6` by calling the `r_scause()` and `r_stval()` functions, respectively. In the original `xv6`, page faults occur only for lazily allocated heap pages created via `sbrk(n, SBRK_LAZY)`, since all other regions (code, data, stack, and heap allocated with `SBRK_EAGER`) are already kept resident in physical memory. 
 
Many CPU architectures support multiple page sizes, including pages significantly larger than the base page size. This support for larger pages (known as *huge pages* or *superpages*) reduces the pressure on the TLB for large allocations while still keeping memory usage at a reasonable level for small allocations. In Sv39, RISC-V supports 2 MiB *megapages* and 1 GiB *gigapages*, each of which must be virtually and physically aligned to a boundary equal to its size. For 2 MiB megapages, the leaf-level (L0) page table is not used; instead, the second-level (L1) page table entry is itself a leaf PTE that points directly to the corresponding 2 MiB page frame. For more information on the RISC-V virtual memory system, please refer to Section 12.4 in the [RISC-V Privileged Architecture](https://csl.snu.ac.kr/courses/4190.307/2026-2/riscv-privileged-20260120.pdf) manual.


### Sv39 RISC-V Page Tables

In Sv39, each user virtual address is translated by walking a three-level page table: L2, L1, and L0.
The physical address of the root page-table page, L2, is stored in the `satp` register, which the hardware uses as the starting point for address translation. Each page-table page occupies one 4 KiB physical frame and contains 512 page-table entries (PTEs), since each PTE is 8 bytes and each page-table index is 9 bits. A non-leaf PTE points to a page table at the next lower level, while a leaf PTE points to a physical data page and stores the permission bits used to control access to that page.

In Sv39, each page table entry (PTE) follows the format shown below.
```
63         54 53                          10  9  8  7   6   5   4   3   2   1   0          
+------------+------------------------------+-----+---+---+---+---+---+---+---+---+
|  Reserved  |  Physical Page Number (PPN)  | RSW | D | A | G | U | X | W | R | V |
+------------+------------------------------+-----+---+---+---+---+---+---+---+---+
```
The RSW field (bits 9-8) is reserved for use by supervisor software. The lowest 8 bits (bits 7-0) have the following meanings.
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

## Problem Specification

### 1. New huge page frame allocator (30 points)

Your first task is to implement the `kalloc_huge()` and `kfree_huge()` functions (@ `./kernel/kalloc.c`) within the `xv6` kernel to allocate or free a 2 MiB chunk of physical memory. Ensure that the starting address of each 2 MiB chunk is aligned to 2 MiB, i.e., the lowest 21 bits of the address must be zero. The new physical memory allocator should be compatible with the existing `kalloc()` and `kfree()` functions, which operate on 4 KiB base pages. Also, you should maximize the number of allocatable 2 MiB frames. For example, allocating 512 4 KiB frames should reduce the number of allocatable 2 MiB frames by at most one. Similarly, freeing 512 4 KiB frames that together fill a 2 MiB-aligned region should make one more 2 MiB frame allocatable.

We only use huge pages for memory-mapped regions. Therefore, these functions are invoked only when `mmap()`-enabled applications are executed in the shell. No existing applications should be affected by the new huge page frame allocator.

__SYNOPSIS__
```
    void *kalloc_huge(void);
```

__DESCRIPTION__

Allocate a 2 MiB huge page frame.

__RETURN VALUE__

* On success, return the start address of the allocated huge page frame. The address should be aligned to a 2 MiB boundary.
* When there is no available huge page frame, return 0.

__SYNOPSIS__
```
    void kfree_huge(void *pa);
```

__DESCRIPTION__

Free the huge page frame that starts at the given address `pa`. If `pa` is not the start address of a previously allocated huge page frame, it should call `panic("kfree_huge")`.

__RETURN VALUE__

* This function does not return any value.

### 2. New `mmap()` and `munmap()` system calls (20 points)

The second task is to implement the `mmap()` and `munmap()` system calls in `kernel/sysproc.c`. We only consider a simplified version of `mmap()`, where only anonymous mapping is supported. `mmap()` reserves a portion of the virtual address space of the calling process, which is backed by physical memory. The physical memory should be allocated *lazily*; when a process calls `mmap()`, the kernel only reserves the virtual memory region without allocating any page frames. 

The first access to a page of a memory-mapped region triggers a page fault whether it is a load or a store. In both cases, the kernel must allocate a page frame, fill it with zeros, and map it with the permissions given in `prot`; a load that first accesses an area mapped by `mmap()` therefore reads zero. Do _not_ satisfy a load fault by mapping a single kernel-wide zero page shared among processes: once a page of a memory-mapped region has been touched, it must be backed by a frame of its own. 

The allocation granularity is one page: a fault in a region mapped without `MAP_HUGEPAGE` allocates the single 4 KiB page containing the faulting address, while a fault in a region mapped with `MAP_HUGEPAGE` allocates the entire 2 MiB huge page containing it. Pages of a region that are never accessed are never allocated. 

__SYNOPSIS__
```
    void *mmap(void *addr, int length, int prot, int flags);
```

__DESCRIPTION__

The `mmap()` system call creates a new mapping in the virtual address space of the calling process. The starting address for the new mapping is specified in `addr`. The `length` argument specifies the length of the mapping in bytes, which must be greater than 0. The actual size of the memory-mapped region is rounded up to a multiple of the base page size (4 KiB) or, when `MAP_HUGEPAGE` is given, of the huge page size (2 MiB). Note that our `mmap()` system call does not support file mapping.

The `prot` argument describes the desired memory protection of the mapping. It is either `PROT_READ` or `PROT_WRITE`.

```
#define PROT_READ      0x0001
#define PROT_WRITE     0x0002

PROT_READ     Pages may be read
PROT_WRITE    Pages may be read and written*
```
 \* In RISC-V, writable pages must also be marked readable. See Section 12.3 in the [RISC-V Privileged Architecture](https://csl.snu.ac.kr/courses/4190.307/2026-2/riscv-privileged-20260120.pdf) manual.
              
The `flags` argument determines whether updates to the mapping are visible to child processes mapping the same region. This behavior is determined by including exactly one of the following values in `flags`.

```
#define MAP_PRIVATE   0x0010
#define MAP_SHARED    0x0020

MAP_PRIVATE   Create a private copy-on-write mapping.
              Updates to the mapping are not visible to child processes mapping the same region.
MAP_SHARED    Share this mapping.
              Updates to the mapping are visible to child processes mapping the same region.
```

In addition, you can combine the following flag with the existing flags using the bitwise OR operation.
```
#define MAP_HUGEPAGE  0x0100

MAP_HUGEPAGE  Allocate the mapping using 2 MiB huge pages.
              When this flag is specified, addr should be aligned to the 2 MiB boundary.
              If this flag is not specified, 4 KiB base pages are used,
              and addr should be aligned to the 4 KiB boundary.
```

__RETURN VALUE__

On success, `mmap()` returns the pointer to the mapped area, which should be the same as `addr`. On error, `mmap()` returns 0 and no mapping is created. In particular, `mmap()` fails if any of the following holds:
* `length` is not greater than 0, or is greater than 64 MiB.
* `addr` is not aligned to the required boundary (2 MiB with `MAP_HUGEPAGE`, 4 KiB otherwise).
* `prot` is neither `PROT_READ` nor `PROT_WRITE`.
* `flags` does not contain exactly one of `MAP_PRIVATE` and `MAP_SHARED`, or contains an undefined bit.
* The requested region overlaps an existing memory-mapped region of the calling process.
* The calling process already has the maximum number of memory-mapped regions, or the system-wide limit has been reached.

The `munmap()` system call simply removes a specified memory-mapped region from the virtual address space. 

__SYNOPSIS__
```
    int munmap(void *addr);
```

__DESCRIPTION__

The `munmap()` system call deletes the mapping that starts at `addr`, and causes further references to addresses within the range to generate invalid memory references. If the associated page frames are no longer in use, it frees them as well. Partial unmapping is not supported: `addr` must be exactly the start address of a region previously returned by `mmap()` in the calling process, and the whole region is unmapped. The memory-mapped regions are also automatically unmapped when the process is terminated. 

__RETURN VALUE__

On success, `munmap()` returns 0. On failure (e.g., `addr` is not the start address of a current mapping of the calling process), it returns -1.

### 3. `mmap()` with `fork()` (30 points)

The `mmap()` system call has two flags, `MAP_PRIVATE` and `MAP_SHARED`, that control the access to memory-mapped regions after `fork()`. When the `MAP_SHARED` flag is specified, the corresponding memory-mapped region is shared by all descendant processes. Therefore, any update to the region is visible to all the processes that share the same region. On the other hand, when the `MAP_PRIVATE` flag is specified, the region can be shared with child processes as long as the region is not updated. However, if a process writes data to the region, a Copy-On-Write (COW) operation should be performed on the affected page. Note that a COW fault on a huge page must copy the whole 2 MiB frame; the region keeps its page size and is never split into base pages.

Note that we don't change anything about the other memory regions, such as code, data, heap, and stack. Code, data, and stack regions remain eagerly allocated: they do not trigger page faults or use huge pages, and their allocated pages are copied to the child during `fork()`. Heap behavior depends on how it is allocated: a normal heap created without `SBRK_LAZY` is also allocated eagerly, while a heap grown with `SBRK_LAZY` is allocated on demand when accessed. Heap pages do not use huge pages. Only memory-mapped regions are affected by the `MAP_PRIVATE` and `MAP_SHARED` flags.

If the kernel cannot obtain a free page frame while handling a page fault, or a process accesses a memory-mapped region in a way that its `prot` does not permit (e.g., a store to a `PROT_READ` region), the offending process should be killed as in the original `xv6`, and the kernel must not panic.

### 4. No memory leaks (10 points)

In order to get full credit for this project, you need to make sure there is no memory leak in your implementation. In the skeleton code, we have added three global variables called `freemem`, `used4k`, and `used2m`. The value of `freemem` indicates the amount of free physical memory in units of 4 KiB, i.e., `freemem` = 100 means 409600 bytes of physical memory are free at that moment. The `used4k` and `used2m` variables represent the number of allocated 4 KiB frames and 2 MiB frames, respectively. You should update these values correctly in your implementation. The physical memory size is configured to 128 MiB in `xv6` by default (`PHYSTOP` @ `kernel/memlayout.h`). Therefore, at any moment, the following equation should hold:
```
freemem = (128 MiB / 4 KiB) - used4k - (512 * used2m)
```
In addition, the skeleton code has a variable named `pagefaults`, which needs to be incremented on every page fault that occurs in a memory-mapped region (see `./kernel/vm.c`).

The current values of these variables are displayed when you press `ctrl-f` on `xv6`. We have also added a system call named `kcall()` (@ `./kernel/sysproc.c`) which returns one of these values as follows:

__SYNOPSIS__
```
#define KC_FREEMEM   0
#define KC_USED4K    1
#define KC_USED2M    2
#define KC_PF        3

int kcall(int n);
```

__DESCRIPTION__

The `kcall()` system call returns the value of `freemem`, `used4k`, `used2m`, or `pagefaults` depending on the argument `n`. If `n` is an undefined value, `kcall()` returns -1.

Whenever you return to the shell after executing a command, the value of `freemem` should remain exactly the same. Otherwise, it means either you forgot to free some page frames or you deallocated some page frames that you shouldn't have.

Before your submission, please make sure your implementation does not have any memory leaks by monitoring the value of `freemem` after running the following programs:

* forktest
* usertests -q
* cat README | wc
* cat README | wc | wc | wc
* cat | cat | cat
* ...

### 5. Design document (10 points)

Prepare a design document detailing your implementation in a single PDF file. Your document should include the following sections.

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones.
   * Explain why these data structures/modifications were necessary and how they contribute to the implementation.
2. Algorithm design
   * Describe the pseudocode for `kalloc_huge()` and `kfree_huge()`, along with a brief description of your allocation/free algorithm for huge pages.
   * Describe the pseudocode for the page fault handler. List all the possible cases you should handle and show what is being done for each case.
   * Describe the pseudocode for `fork()`. List all the possible cases you should handle.
   * Describe all corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
3. Testing and validation
   * Outline the test cases you created to validate your implementation, if any.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.
   * Explain which part of the project consumed most of your time and why.

## Restrictions

* You can assume the following:
  * The valid range of the target virtual address (`addr`) in `mmap()` is from `PHYSTOP` to `MAXVA - 0x10000000`.
  * The maximum size (`length`) in `mmap()` is limited to 64 MiB.
  * Each process can have up to 4 memory-mapped regions.
  * The system can support up to 64 distinct memory-mapped regions in total.
* On `exit()` or `exec()`, all the memory-mapped regions should be unmapped.
* Your implementation should work on multi-core systems. The `CPUS` variable that represents the number of CPUs in the target QEMU machine emulator is already set to 4 in the `Makefile`.
* Do not add any other system calls.
* You only need to modify the files in the `./kernel` directory. Do not modify `./kernel/ktest.c`; changes to it and to any file outside `./kernel` will be ignored during grading.
  
## Skeleton code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ cd xv6-riscv-snu
$ git fetch --all
$ git checkout pa4
```
After downloading, you have to set your `STUDENTID` in the `Makefile` again.

To help with debugging and automatic grading, we have added another system call named `ktest()` (@ `./kernel/ktest.c`) and the test program called `ktest` (@ `./user/ktest.c`). The `ktest()` system call is used to invoke the `kalloc()`, `kfree()`, `kalloc_huge()`, and `kfree_huge()` kernel functions directly.

__SYNOPSIS__
```
#define KT_KALLOC       0
#define KT_KFREE        1
#define KT_KALLOC_HUGE  2
#define KT_KFREE_HUGE   3

void *ktest(int n, void *pa);
```

__DESCRIPTION__

The `ktest()` system call invokes one of the `kalloc()`, `kfree()`, `kalloc_huge()`, and `kfree_huge()` functions and passes back its return value. For `KT_KALLOC` and `KT_KALLOC_HUGE`, the second argument `pa` is ignored. For `KT_KFREE` and `KT_KFREE_HUGE`, `pa` indicates the address previously allocated by either `ktest(KT_KALLOC, 0)` or `ktest(KT_KALLOC_HUGE, 0)`. Note that `ktest(KT_KALLOC, 0)` and `ktest(KT_KALLOC_HUGE, 0)` return an address that belongs to the kernel address space, although `ktest()` is called by user applications. So, those addresses cannot be used directly in user space to access memory.

__RETURN VALUE__

* On success, `ktest(KT_KALLOC, 0)` and `ktest(KT_KALLOC_HUGE, 0)` will return the allocated kernel virtual address. If the allocation fails, they will return `(void *) 0`. 
* `ktest(KT_KFREE, pa)` and `ktest(KT_KFREE_HUGE, pa)` always return `(void *) 0`.

In addition, the `pa4` branch includes three user-level programs (`mmaptest1`, `mmaptest2`, and `mmaptest3`) whose source code is available in `./user/mmaptest1.c`, `./user/mmaptest2.c`, and `./user/mmaptest3.c`, respectively. These programs test various `mmap()` scenarios.

## Sample Outputs

The following shows the sample output for each test program provided in the skeleton code.

### 1. `ktest`

The `ktest` program displays the physical memory allocation status after executing `kalloc()`, `kfree()`, `kalloc_huge()`, and `kfree_huge()` consecutively. You can observe that the values of `freemem`, `used4k`, and `used2m` change accordingly. Note that the initial `freemem` value (32553) may vary depending on your specific implementation.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 3 starting
hart 1 starting
hart 2 starting
init: starting sh
$ ktest
Init: freemem=32553, used4k=215, used2m=0
After kalloc(): freemem=32552, used4k=216, used2m=0
After kfree(): freemem=32553, used4k=215, used2m=0
After kalloc_huge(): freemem=32041, used4k=215, used2m=1
After kfree_huge(): freemem=32553, used4k=215, used2m=0
$
```

### 2. `mmaptest1`

The `mmaptest1` program simply creates a memory-mapped region using the `mmap()` system call, writes the value `0xdeadbeef` at the start of this region, and subsequently reads and prints this value.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 3 starting
hart 1 starting
init: starting sh
$ mmaptest1
pid 3: DEADBEEF
$
```

### 3. `mmaptest2`

In `mmaptest2`, the parent process (pid 3) creates a *shared* memory-mapped region backed by a 2 MiB huge page (`MAP_SHARED | MAP_HUGEPAGE`) and writes the value `0xdeadbeef` at the start of the region. It then forks a child process (pid 4), which overwrites the value `0x0badbeef` at the same location and reads it back. Since the memory-mapped region is shared between the parent and child processes (`MAP_SHARED`), both processes should read the same value `0x0badbeef`. If the parent process had initially mapped the region with `MAP_PRIVATE`, it would read the value `0xdeadbeef`. While running this program, the value of `used2m` should be 1, since the whole region is backed by a single 2 MiB frame.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
hart 3 starting
init: starting sh
$ mmaptest2
After mmap(): used2m=0
After store: used2m=1
pid 4: BADBEEF
pid 3: BADBEEF
$
```

### 4. `mmaptest3`

In `mmaptest3`, the parent process (pid 3) creates a memory-mapped region with the `MAP_PRIVATE` flag and writes the value `0xdeadbeef` at the beginning of the region _before_ forking, so that the region is already backed by a page frame. Afterward, it forks a child process (pid 4), and the child process further forks a grandchild process (pid 5). At this point the frame is shared by all three processes, so when the grandchild process writes the value `0x0badbeef` at the beginning of the memory-mapped region, a Copy-On-Write operation must take place. Since the memory-mapped region is private to each process due to the use of `MAP_PRIVATE`, only the grandchild process reads the value `0x0badbeef`. Meanwhile, both the parent (pid 3) and the child process (pid 4) read the original value `0xdeadbeef`. If the parent process had initially mapped the region with `MAP_SHARED`, then all the processes would read the same value `0x0badbeef`.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 3 starting
hart 2 starting
hart 1 starting
init: starting sh
$ mmaptest3
pid 5: BADBEEF
pid 4: DEADBEEF
pid 3: DEADBEEF
$
```

## Tips

* Read Chap. 3 -- 5 of the [xv6 book](https://mit-pdos.github.io/xv6-riscv-book/book.pdf) to understand the virtual memory subsystem and page-fault exceptions in `xv6`.

* Whenever you modify a valid PTE, call `sfence_vma()` (@ `./kernel/riscv.h`) before returning to user mode. Otherwise, the processor may continue using a stale TLB entry and ignore your updated PTE permissions or physical address.

* Remember that `fork()` and the page fault handler may run concurrently on different harts. Protect any shared bookkeeping (e.g., reference counts for shared or COW frames) with a lock.

* For your reference, the following roughly shows the required code changes; each `+` denotes about 1~10 lines to add, remove, or modify.
   ```
   kernel/proc.h        |  +
   kernel/riscv.h       |  +
   kernel/kalloc.c      |  ++++++++++++
   kernel/vm.c          |  +++++++++++++++++++++++++++++
   kernel/proc.c        |  +
   kernel/sysproc.c     |  ++++
   ```

## Hand in instructions

* First, make sure you are on the `pa4` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the parent directory. Upload this file to the submission server, together with your design document as a single PDF file.
  
* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button.
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Adding your new IP address is automated by a script that periodically checks the Google Form every 10 minutes.
     + If you cannot reach the server 10 minutes after submitting the Google Form, send the request again, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, that is likely due to an error in the automated process. The TAs will check if the script is properly running, but that is a ___manual___ process, so please do not expect it to be completed immediately.


## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive full credit. 25% of the credit will be deducted for every single day delayed.
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.


Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)



