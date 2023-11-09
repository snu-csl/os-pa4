# 4190.307 Operating Systems (Fall 2023)
# Project #4: mmap() with Huge Pages
### Due: 11:59 PM, November 25 (Saturday)

## Introduction

In this project, we implement a simplified version of the `mmap()` system call using  2MiB huge pages supported by the RISC-V architecture. The goal of this project is to understand the paging hardware of the RISC-V processor and how the operating system manages virtual-to-physical address mapping to improve memory efficiency.

## Background

### `mmap()` system call

In UNIX-based systems, the `mmap()` system call creates a new mapping in the virtual address space of the calling process. In *file-backed mapping*, `mmap()` maps an area of the process's virtual address to files, allowing file I/O to be treated as memory access. On the other hand, in *anonymous mapping*, `mmap()` maps an area of the process's virtual memory not backed by any file, with the contents being initialized to zero. In this project, we focus on the `mmap()` system call that only supports anonymous mapping. For additional details on `mmap()`, please refer to its manual page using the `$ man 2 mmap` command. 

### RISC-V paging hardware

The RISC-V processor implements a standard paging with a (base) page size of 4KiB. Especially, `xv6` runs on Sv39 RISC-V, which uses 39-bit virtual addresses with three-level page tables. When the paging is turned on in the supervisor mode, the `satp` register points to the physical address of the root page table. A page fault occurs when a process attempts to access a page with an invalid PTE (page table entry) or with an invalid access permission. 

When a trap is taken into the supervisor mode, the `scause` register is written with a code indicating the event that caused the trap as shown in the following table.

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
| 12              | Instruction page fault      |
| 13              | __Load page fault__             |
| 14              | _Reserved_                      |
| 15              | __Store page fault__        |
| >= 16           | _Reserved_                      |

Please pay attention to the following two events in the above table: Load page fault (13) and Store page fault (15). These events indicate page faults caused by load instructions, or store instructions, respectively. On a page fault, RISC-V also provides the information on the _virtual_ address that caused the fault using the `stval` register. Currently, no page faults occur in `xv6` because all the required code and data are resident in the physical memory. However, you will need to handle these page faults in this project. Note that the value of the `scause` or `stval` registers can be read by calling the `r_scause()` or `r_stval()` function in `xv6`, respectively. 
 
Many CPU architectures support multiple page sizes, including pages significantly larger than the base page size. This support for larger pages (known as *huge pages* or *superpages*) reduces the pressure on the TLB for large allocations while still keeping memory usage at a reasonable level for small allocations. In Sv39, RISC-V supports 2MiB *megapages* and 1GiB *gigapages*, each of which must be virtually and physically aligned to a boundary equal to its size. For 2MiB megapages, the leaf page table is not used; instead, the second-level page table entry directly points to the location of the corresponding 2MiB page frame. For more information on the RISC-V's virtual memory system, please refer to Section 4.4 in the [RISC-V Privileged Architecture](http://csl.snu.ac.kr/courses/4190.307/2023-2/riscv-privileged-20211203.pdf) manual.

## Problem specification

### 1. New huge page frame allocator (30 points)

Your first task is to implement `kalloc_huge()` and `kfree_huge()` functions (@ `./kernel/kalloc.c`) within the `xv6` kernel to allocate or free a 2MiB chunk of physical memory. Ensure that the starting address of each 2MiB chunk is aligned to 2MiB, i.e., the last 21 bits of the address must be zero. The new physical memory allocator should be compatible with the existing `kalloc()` and `kfree()` functions, which operate with 4KiB base page sizes. Also, you should maximize the number of allocatable 2MiB frames. For example, allocating 512 4KiB frames should reduce the number of allocatable 2MiB frames by at most one. Similarly, freeing 512 4KiB frames that fit into a 2MiB frame should allow for an additional allocation of a 2MiB frame.

We only use huge pages for memory-mapped regions. Therefore, these functions are invoked only when `mmap()`-enabled applications are executed in the shell. No existing applications should be affected by the new huge page frame allocator.

__SYNOPSIS__
```
    void *kalloc_huge();
```

__DESCRIPTION__

Allocate a 2MiB huge page frame.

__RETURN VALUE__

* On success, return the start address of the allocated huge page frame. The address should be aligned to the 2MiB boundary.
* When there is no available huge page frame, return 0.

__SYNOPSIS__
```
    void kfree_huge(void *pa);
```

__DESCRIPTION__

Free the corresponding huge page frame that starts with the given address `pa`. If `pa` is not the address of the previously allocated huge page frame, it should call `panic("kfree_huge")`.

__RETURN VALUE__

* This function does not return any value.

### 2. New `mmap()` and `munmap()` system calls (20 points)

The second task is to implement `mmap()` and `munmap()` system calls in `kernel/sysproc.c`. We only consider a simplified version of `mmap()`, where only anonymous mapping is supported. Basically, `mmap()` reserves a portion of the virtual memory of the calling process. Those virtual memory regions are backed by physical memory. The physical memory should be allocated *lazily*; when a process calls `mmap()`, the kernel only reserves the virtual memory region without allocating any page frames. Later, a load or store instruction targeting the memory-mapped region will trigger a page fault, prompting the kernel to allocate the relevant page frame. When a load instruction is initially performed on an area mapped by `mmap()`, it should return zero. 

The `mmap()` system call supports both 4KiB base pages and 2MiB huge pages. When the `MAP_HUGEPAGE` flag is specified, 2MiB huge page frames are allocated. Otherwise, 4KiB base page frames are allocated.
`munmap()` simply removes a specified memory-mapped region from the virtual address space. 

__SYNOPSIS__
```
    void *mmap(void *addr, int length, int prot, int flags);
```

__DESCRIPTION__

The `mmap()` system call creates a new mapping in the virtual address space of the calling process. The starting address for the new mapping is specified in `addr`. The `length` argument specifies the length of the mapping (which must be greater than 0), in bytes.
If another mapping already exists in the specified region of the virtual address space, the kernel will return 0. Otherwise, the kernel returns the original address `addr`. The actual size of the memory-mapped region is rounded up to the base/huge page size. Note that our `mmap()` system call does not support file mapping. 

The `prot` argument describes the desired memory protection of the mapping. It is either `PROT_READ` or `PROT_WRITE`.

```
#define PROT_READ      0x0001
#define PROT_WRITE     0x0002

PROT_READ     Pages may be read
PROT_WRITE    Pages may be read and written*
```
 \* In RISC-V, writable pages must also be marked readable. See Section 4.3.1 in the [RISC-V Privileged Architecture](http://csl.snu.ac.kr/courses/4190.307/2023-2/riscv-privileged-20211203.pdf) manual.
              
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

MAP_HUGEPAGE  Allocate the mapping using 2MiB huge pages.
              When this flag is specified, addr should be aligned to the 2MiB boundary.
              If this flag is not specified, 4KiB base pages are used
              and addr should be aligned to the 4KiB boundary.
```

__RETURN VALUE__

On success, `mmap()` returns the pointer to the mapped area which should be the same as `addr`. On error, `mmap()` returns 0. 


__SYNOPSIS__
```
    int munmap(void *addr);
```

__DESCRIPTION__

The `munmap()` system call deletes the mapping that starts with `addr`, and causes further references to address within the range to generate invalid memory references. If the associated page frames are no longer in use, it frees them as well. The memory-mapped regions are also automatically unmapped when the process is terminated. 

__RETURN VALUE__

On success, `munmap()` returns 0. On failure, it returns -1.

### 3. `mmap()` with `fork()` (30 points)

The `mmap()` system call has two flags, `MAP_PRIVATE` and `MAP_SHARED`, that control the access to memory-mapped regions after `fork()`. When the `MAP_SHARED` flag is specified, the corresponding memory-mapped region is shared by all the child processes. Therefore, any update to the region is visible to all the processes that share the same region. On the other hand, when the `MAP_PRIVATE` flag is specified, the region can be shared with child processes as long as the region is not updated. However, if a process writes data to the region, the affected page should be copy-on-writed. 

Note that we don't change anything about the other memory regions, such as code, data, heap, and stack. They are NOT demand-loaded, i.e., no page faults occur in those regions, and they don't use huge pages. The page frames associated with those regions are simply copied to the child process during `fork()`. Only the memory-mapped regions are affected by `MAP_PRIVATE` or `MAP_SHARED` flag.

### 4. No memory leaks (10 points)

In order to get full credit for this project, you need to make sure there is no memory leak in your implementation. In the skeleton code, we have added three global variables called `freemem`, `used4k`, and `used2m`. The value of `freemem` indicates the amount of free physical memory in 4KiB, i.e., `freemem` = 100 denotes that 409600 bytes of physical memory are free at that moment. The `used4k` and `used2m` variables represent the number of allocated 4KiB frames and 2MiB frames, respectively. You should update these values correctly in your implementation. The physical memory size is configured to 128MiB in `xv6` by default (`PHYSTOP` @ `kernel/memlayout.h`). Therefore, at any moment, the following equation should hold:
```
freemem = (128MiB / 4KiB) - used4k - (512 * used2m)
```
In addition, the skeleton code has a variable named `pagefaults`, which is incremented by one on every page fault (see `pagefault()` @ `./kernel/proc.c`). 

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

The `kcall()` system call returns the value of `freemem`, `used4k`, `used2m`, and `pagefaults` depending on the argument `n`. If `n` is an undefined value, `kcall()` returns -1.

Whenever you return to the shell after executing a command, the value of `freemem` should remain exactly the same. Otherwise, it means either you forgot to free some page frames or you deallocated some page frames that you shouldn't. 

Before your submission, please make sure your implementation does not have any memory leaks by monitoring the value of `freemem` after running the following programs:

* forktest
* usertests
* cat README | wc
* cat README | wc | wc | wc
* cat | cat | cat
* ...

### 5. Design document (10 points)

You need to prepare and submit the design document (in a single PDF file) for your implementation. Your design document should include the following:

* New data structures you have introduced and why
* The pseudocode for `kalloc_huge()` and `kfree_huge()`, along with a brief description of your allocation/free algorithm for huge pages
* The pseudocode for the page fault handler. List all the possible cases you should handle and show what is being done for each case.
* The pseudocode for `fork()`. List all the possible cases you should handle and show what is being done for each case. 

## Restrictions

* You can assume the following:
  * The valid range of the target virtual address (`addr`) in `mmap()` is from `PHYSTOP` to `MAXVA - 0x10000000`.
  * The maximum size (`length`) in `mmap()` is limited to 64MiB.
  * Each process can have up to 4 memory-mapped regions.
  * The system can support up to 64 distinct memory-mapped regions in total.
* On `exit()` or `exec()`, all the memory-mapped regions should be unmapped.
* Your implementation should work on multi-core systems. The `CPUS` variable that represents the number of CPUs in the target QEMU machine emulator is already set to 4 in the `Makefile`.
* Do not add any other system calls.
* You only need to modify those files in the `./kernel` directory (except for the `./kernel/ktest.c` file). Changes to other files will be ignored during grading. 

## Skeleton code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa4
```
After downloading, you have to set your `STUDENTID` in the `Makefile` again.

To help debugging and automatic grading, we have added another system call named `ktest()` (@ `./kernel/ktest.c`) and the test program called `ktest` (@ `./user/ktest.c`). The `ktest()` system call is used to invoke the `kalloc()`, `kfree()`, `kalloc_huge()`, and `kfree_huge()` kernel functions directly. 

__SYNOPSIS__
```
#define KT_KALLOC       0
#define KT_KFREE        1
#define KT_KALLOC_HUGE  2
#define KT_KFREE_HUGE   3

    void *ktest(int n, void *pa);
```

__DESCRIPTION__

The `ktest()` system call invokes one of `kalloc()`, `kfree()`, `kalloc_huge()`, and `kfree_huge()` functions and passes its return value. For `KT_KALLOC` and `KT_KALLOC_HUGE`, the second argument `pa` is ignored. For `KT_KFREE` and `KT_KFREE_HUGE`, `pa` indicates the address previously allocated by either `ktest(KT_KALLOC, 0)` or `ktest(KT_KALLOC_HUGE, 0)`. Note that `ktest(KT_KALLOC, 0)` or `ktest(KALLOC_HUGE, 0)` returns the address that belongs to the kernel address space, although `ktest()` is called by user applications. So, those addresses cannot be used directly in the user space to access memory.

__RETURN VALUE__

* On success, `ktest(KALLOC)` and `ktest(KALLOC_HUGE)` will return the allocated kernel virtual address. If the allocation fails, they will return `(void *) 0`. 
* `ktest(KFREE)` and `ktest(KFREE_HUGE)` will return `(void *) 0` all the time.

In addition, the `pa4` branch includes three user-level programs (`mmaptest1`, `mmaptest2`, and `mmaptest3`) whose source code is available in `./user/mmaptest1.c`, `./user/mmaptest2.c`, and `./user/mmaptest3.c`, respectively. These programs test various scenarios associated with `mmap()`. 

## Sample Outputs

The following shows the sample output of each test program provided in the skeleton code.

### 1. `ktest`

The `ktest` program displays the physical memory allocation status after executing `kalloc()`, `kfree()`, `kalloc_huge()`, and `kalloc_free()` consecutively. You can observe that the values of `freememe`, `used4k`, and `used2m` change accordingly. Note that the initial `freemem` value (32553) may vary depending on your specific implementation.

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

In `mmaptest2`, the parent process (pid 3) creates a *shared* memory-mapped region and writes the value `0xdeadbeef` at the start of the region. Subsequently, it forks a child process (pid 4), which overwrites the value `0x900dbeef` in the same location. Since the memory-mapped region is shared between the parent and child processes (`MAP_SHARED`), both processes should read the same value `0x900dbeef`. If the parent process initially mapped the region with `MAP_PRIVATE`, it would read the value `0xdeadbeef`.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 1 starting
hart 2 starting
hart 3 starting
init: starting sh
$ mmaptest2
pid 4: 900DBEEF
pid 3: 900DBEEF
$
```

### 4. `mmaptest3`

In `mmaptest3`, the parent process (pid 3) creates a memory-mapped region with the `MAP_PRIVATE` flag. Afterwards, it forks a child process (pid 4) and the child process further forks a grandchild process (pid 5). The grandchild process writes the value `0x900dbeef` at the beginning of the memory-mapped region. Since the memory-mapped region is private to each process due to the use of `MAP_PRIVATE`, only the grandchild process reads the value `0x900dbeef`. Meanwhile, both the parent (pid 3) and the child process (pid 4) read the value `0`. If the parent process initially mapped the region with `MAP_SHARED`, then all the processes would read the same value `0x900dbeef`.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 4 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 3 starting
hart 2 starting
hart 1 starting
init: starting sh
$ mmaptest3
pid 5: 900DBEEF
pid 4: 0
pid 3: 0
```

## Tips

* Read Chap. 3 and 4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2023-2/book-riscv-rev3.pdf) to understand the virtual memory subsystem and page-fault exceptions in `xv6`. 
  
## Hand in instructions

* First, make sure you are on the `pa4` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file to the submission server.
* Please remove all the debugging outputs before you submit.
* The total number of submissions for this project assignment will be limited to 30. Only the version marked `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Note that adding your new IP address is a ___manual___ process, so please do not expect it to be completed immediately.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delay.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server right after each submission. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
