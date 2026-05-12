# 4190.307 Operating Systems (Spring 2026)
# Project #4: Holy COW!
### Due: 11:59 PM, May 31 (Sunday)

## Introduction

COW (Copy-on-Write) is a virtual memory technique that lets multiple address spaces share the same physical memory until one of them tries to modify it. Instead of eagerly copying all user memory during `fork()`, the kernel marks shared pages read-only and creates a private copy only on a write fault.
In this project, you will implement COW in `xv6` in two stages: first for user pages, and then for leaf page-table pages as well. The goal of this project is to understand the `xv6` virtual memory subsystem.

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

### Sv39 RISC-V Page Tables

In Sv39, each user virtual address is translated by walking a three-level page table: L2, L1, and L0.
The physical address of the root page-table page, L2, is stored in the `satp` register, which the hardware uses as the starting point for address translation. 
Each page-table page occupies one 4KiB physical frame and contains 512 page-table entries (PTEs), since each PTE is 8 bytes and each page-table index is 9 bits. A non-leaf PTE points a page table at the next lower level, while a leaf PTE points to a physical data page and stores the permission bits used to control access to that page.

In Sv39, each page table entry (PTE) follows the format shown below.
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

## Problem Specification

### Part 1. `ptpages()` System Call (10 points)

First, you must implement the `ptpages()` system call. Its system call number is assigned to 40 in `kernel/syscall.h`.

__NAME__

* `ptpages() ` -- return the number of user page-table pages allocated at a specified page-table level.

__SYNOPSYS__
```C
    int ptpages(int level);
```

__DESCRIPTION__

The `ptpages()` system call returns the number of page-table pages currently allocated for all user address spaces at the requested page-table level.
The valid levels are:

| Argument | Meaning                  |
|:--------:|:-------------------------|
| `0`      | L0 leaf page-table pages |
| `1`      | L1 page-table pages      |
| `2`      | L2 root page-table pages |

The kernel internally maintains an array, `int ptpages[3]`, to track the number of user page-table pages allocated at each level. Specifically, `ptpages[i]` records the number of page-table pages at level `i`. Each counter must be incremented whenever a user page-table page at the corresponding level is allocated, and decremented whenever such a page is freed. Page-table pages used only by the kernel must not be included in these counters.

This system call will be used in Part 3 to check whether sharing L0 page tables actually reduces the number of allocated L0 page-table pages.

__RETURN VALUE__

* `ptpages(level)` returns the requested page-table-page count on success.
* It returns -1 if `level` is not 0, 1, or 2.

### Part 2. COW for User Address-Space Pages (30 points)

In modern operating systems, process creation via `fork()` is commonly optimized using the Copy-on-Write (COW) mechanism. Instead of copying all of the parent's physical memory into the child, the kernel initially allows the two processes to share the same physical pages. The actual copy is delayed until one of the processes attempts to modify a shared page.

In Part 2, you will implement COW for user address-space pages. When `fork()` creates a child process, the child should initially share the parent's user memory pages rather than receiving eager copies of them. For each valid, user-accessible, writable page in the parent, the kernel should map the same physical page at the same virtual address in the child, clear the `PTE_W` bit in both parent and child PTEs, and mark both mappings as COW using a software-defined PTE bit. This makes the shared page read-only from the hardware's point of view, while allowing the kernel to distinguish intentional COW mappings from genuinely read-only pages.

Code pages are already read-only and may be shared between the parent and child, but they should not be marked as COW because they are not supposed to become writable. Likewise, the trapframe and trampoline pages are special kernel-related mappings and are not targets of COW in this project.

After `fork()`, both parent and child may continue reading shared pages without copying. If either process attempts to write to a COW page, the write will cause a store page fault. The page-fault handler must recognize that the faulting address refers to a valid COW mapping, allocate a new physical page, copy the old page's contents into it, and update only the faulting process's PTE to point to the new page with write permission restored and the COW bit cleared. The kernel must ensure that a physical page is not freed while it is still shared. 

_Parent process (Before fork())_

```
  L2 page table           L1 page table           L0 page table               Data page
+---------------+   +-->+---------------+   +-->+---------------+   +---->+---------------+ 
| 0x2000 |    V |---|   | 0x3000 |    V |---|   | 0x4000 | R XV |   |     |               |
+---------------+       +---------------+       +---------------+   |     |               |
|               |       |               |       | 0x5000 | RW V |---|     |               |
|               |       |               |       +---------------+         |               |
|               |       |               |       |               |         |               |
+---------------+       +---------------+       +---------------+         +---------------+
```

_Parent and child processes (After fork())_

```
  L2 page table           L1 page table           L0 page table               Data page
+---------------+   +-->+---------------+   +-->+---------------+   +--+->+---------------+
| 0x2000 |    V |---|   | 0x3000 |    V |---|   | 0x4000 | R XV |   |  |  |               |
+---------------+       +---------------+       +---------------+   |  |  |               |
|               |       |               |       | 0x5000 |CR  V |---|  |  |               |
|               |       |               |       +---------------+      |  |               |
|               |       |               |       |               |      |  |               |
+---------------+       +---------------+       +---------------+      |  +---------------+
                                                                       |
                                                                       |
                                                                       |
  L2 page table           L1 page table           L0 page table        |       
+---------------+   +-->+---------------+   +-->+---------------+      |       
| 0xA000 |    V |---|   | 0xB000 |    V |---|   | 0x4000 | R XV |      |    
+---------------+       +---------------+       +---------------+      |     
|               |       |               |       | 0x5000 |CR  V |------+     
|               |       |               |       +---------------+            
|               |       |               |       |               |            
+---------------+       +---------------+       +---------------+
* C means a COW mapping.         
```

### Part 3. COW for Leaf (L0) Page Tables (50 points)

In Part 3, you will extend your Part 2 implementation so that `fork()` can share leaf (L0) page-table pages between the parent and child when possible. In Part 2, the parent and child share physical memory pages, but the child still needs to set up its own page-table pages.
Part 3 reduces this additional memory overhead by allowing the parent and child to share the L0 page-table pages themselves.
Since each L0 page-table page contains 512 PTEs and can describe up to 2MiB of virtual address space, sharing these pages can substantially reduce memory usage for programs that repeatedly fork processes with large or sparse address spaces.

With this optimization, the child's page-table structure may refer to the same L0 page-table page used by the parent. The mappings inside that shared L0 page table still refer to the shared physical memory pages from Part 2. Therefore, after `fork()`, the parent and child may share both the leaf page-table page and the physical pages described by its PTEs.

However, sharing a page-table page introduces an important constraint: a process must not directly modify a PTE inside an L0 page table that is still shared with another process. If the kernel needs to change a user PTE and that PTE resides in a shared L0 page-table page, the kernel must first create a private copy of the L0 page-table page for the current process. Only after the process has its own private L0 page table, it is safe to update the desired PTE. This rule is necessary because modifying a shared L0 page table would otherwise change the address spaces of multiple processes at the same time.

Your implementation must distinguish between sharing a page-table page and sharing a physical memory page. These are related but separate forms of sharing.
A private L0 page table does not necessarily mean that all physical pages referenced by that L0 page table are private. Copying an L0 page table copies PTEs, not the physical pages to which those PTEs point. 

For this reason, your Part 3 implementation must preserve the correctness guarantees from Part 2. User address-space pages must still be protected by COW, writes must still create private copies when necessary, and physical pages must not be freed while they are still referenced by another process. The implementation should also avoid treating trapframe or trampoline pages as COW targets, since they are special mappings rather than ordinary user address-space pages.

_Parent process (Before fork())_

```
  L2 page table           L1 page table           L0 page table               Data page
+---------------+   +-->+---------------+   +-->+---------------+   +---->+---------------+
| 0x2000 |    V |---|   | 0x3000 |    V |---|   | 0x4000 | R XV |   |     |               |
+---------------+       +---------------+       +---------------+   |     |               |
|               |       |               |       | 0x5000 | RW V |---|     |               |
|               |       |               |       +---------------+         |               |
|               |       |               |       |               |         |               |
+---------------+       +---------------+       +---------------+         +---------------+
```

_Parent and child processes (After fork())_

```
  L2 page table           L1 page table           L0 page table               Data page
+---------------+   +-->+---------------+   +-->+---------------+   +---->+---------------+
| 0x2000 |    V |---|   | 0x3000 |    V |---|   | 0x4000 | R XV |   |     |               |
+---------------+       +---------------+   |   +---------------+   |     |               |
|               |       |               |   |   | 0x5000 |CR  V |---|     |               |
|               |       |               |   |   +---------------+         |               |
|               |       |               |   |   |               |         |               |
+---------------+       +---------------+   |   +---------------+         +---------------+
                                            |                           
                                            |                           
                                            |                           
  L2 page table           L1 page table     |                    
+---------------+   +-->+---------------+   |  
| 0xA000 |    V |---|   | 0x3000 |    V |---|   
+---------------+       +---------------+         
|               |       |               |         
|               |       |               |             
|               |       |               |          
+---------------+       +---------------+
* C means a COW mapping.               
```

### Part 4. Design Document (10 points)

Prepare a design document detailing your implementation in a single PDF file. Your document should include the following sections.

1. New data structures
   * Provide details about any newly introduced data structures or modifications made to existing ones.
   * Explain why these data structures/modifications were necessary and how they contribute to the implementation.
3. Algorithm design
   * Explain how you maintain reference counts for shared physical pages and shared L0 page-table pages
   * Explain how `fork()` works in Part 2 and Part 3
   * Explain how your trap handler resolves page faults in Part 2 and Part 3
   * Explain how your implementation handles page-table sharing in Part 3, including when an L0 page-table page must be copied
   * Explain which system calls or kernel paths are affected by Part 2 and Part 3, and why.
   * Describe all corner cases you considered and the strategies you used to address them.
   * Discuss any optimizations you applied to improve code efficiency, both in terms of time and space.
5. Testing and validation
   * Outline the test cases you created to validate your implementation.
   * Describe how you verified the correct handling of the corner cases mentioned in Section 2.
   * Explain which part of the project consumed most of your time and why.

### Correctness Requirement: Multi-hart Support without Memory Leaks

You must make your implementation correct on multi-hart systems. Your COW implementation from Part 2 and 3 should be race-free and deadlock-free when multiple harts execute kernel code concurrently. Any shared state must be protected consistently so that concurrent page faults, forks, exits, and address-space modifications do not corrupt memory or page tables. 

Your implementation must also have no memory leaks. All page frames and kernel allocations must be freed on normal execution paths as well as on error and rollback paths. At the same time, the kernel must avoid double frees, especially when physical pages or L0 page-table pages are shared by multiple processes.

The skeleton code provides a global variable named `freemem` to help detect memory leaks. This counter is decremented exactly once on each successful `kalloc()` and incremented exactly once on the corresponding `kfree()`. This accounting logic is already present in the skeleton code, and you should not modify it. Whenever control returns to `sh` after running a user program, the value of `freemem` should match its value before the program started.
  
The `ptpages[]` counters must also be restored correctly. After a user program finishes and control returns to `sh`, the values of `ptpages[0]`, `ptpages[1]`, and `ptpages[2]` should match their values before the program started. These counters help detect leaks of user page-table pages at each level. The current values of `freemem` and `ptpages[]` can be displayed by pressing `ctrl`+`p` (`^p`) on the xv6 console.

Full credit for Part 2 and 3 requires passing quick usertests (`$ usertests -q`) on a multi-hart configuration (e.g., `CPUS` > 1 in `Makefile`), with no memory leaks reported by either the `freemem` counter or the `ptpages[]` counters.


### BONUS: Memory-efficiency (up to an additional 10 points)

In addition to the required parts above, we will award bonus points for memory-efficient COW management.

Submissions will be ranked by the amount of memory used by the kernel for COW-related metadata and page-table management. 
Specifically, we will consider both the value of `freemem` when the initial `sh` process starts and the minimum value of `freemem` observed while running a benchmark that stresses the Part 3 implementation. This prevents implementations from appearing memory-efficient at boot time while allocating large COW metadata structures later during execution.

Bonus points will be awarded as follows:
* The top 10% of eligible submissions will receive a 10% bonus.
* The next 10% of eligible submissions will receive a 5% bonus.

Only submissions that pass the required tests are eligible for the bonus. Implementations must remain correct, race-free, and leak-free. Reducing memory usage by breaking correctness, omitting required functionality, or hiding allocations from the accounting mechanism will not be considered valid for the bonus.
 
## Restrictions

* For Part 2, we will run your code with the `-DPART2` compiler flag. Therefore, please enclose any code modifications for Part 2 within the `#ifdef PART2` and `#endif` macros.
  
* For Part 3, we will run your code only with the `-DPART3` compiler flag.

* Please use `qemu` version 8.2.0 or later. To check your `qemu` version, run: `$ qemu-system-riscv64 --version`

* You are required to modify only the files in the `./kernel` directory. Any other changes will be ignored during grading.

## Tips

* You can repurpose the RSW field (PTE bits 9-8) to store software state. For example, use one bit to mark that a PTE is a COW mapping.

* Whenever you modify a valid PTE, call `sfence_vma()` before returning to user mode. Otherwise, the processor may continue using a stale TLB entry and ignore your updated PTE permissions or physical address.
 
* Read Chap. 3 and 5 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2026-1/book-riscv-rev5.pdf) to understand the virtual memory subsystem and page-faults in `xv6`.
  
* For your reference, the following roughly shows the required code changes; each `+` denotes about 1~10 lines to add, remove, or modify.
   ```
   kernel/riscv.h       |  +
   kernel/kalloc.c      |  +++++
   kernel/vm.c          |  ++++++++++++++++
   kernel/proc.c        |  +
   ```

## Skeleton Code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ cd xv6-riscv-snu
$ git checkout pa4
```
After downloading the skeleton code, you must first set your `STUDENTID` in the `Makefile` again.

The skeleton code has also been modified so that pressing ctrl-p (`^p`) at the shell prompt prints the current values of `feemem` and `ptpages[]`.
The `freemem` counter is already maintained by the skeleton code. The skeleton includes `ptpages[]` counters, but it does not maintain them correctly. Updating `ptpages[]` whenever page-table pages are allocated or freed should be implemented in Part 1.

## Example: `cowtest` 

The `./user` directory contains a test program named `cowtest.c`. This program is designed to demonstrate the effect of combining copy-on-write data pages with shared L0 page tables.

The program allocates a 20 MiB heap region aligned to a 2 MiB boundary. Since each L0 page table covers 2 MiB of virtual address space, this heap region spans exactly 10 L0 page tables. The parent process first populates the heap so that all 20 MiB of heap pages are physically allocated. Then it forks 10 child processes. Each child writes to one page in each 2 MiB region of the heap, then exits. At the end of each phase, the parent prints the number of free page frames (`freemem`) and the number of allocated page-table pages at each level (`ptpages[]`).

The following shows a sample output. Note that the initial value of `freemem` can vary depending on your implementation.

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ ^p
1 sleep  init
2 sleep  sh
freemem=32512                  <-- freemem before running any program
ptpages=[L2:2, L1:4, L0:4]     <-- ptpages[] before running any program

$ cowtest
cowtest: heap size=20 MiB NL0=10 NCHILD=10
freemem=32497 ptpages[L2=3 L1=6 L0=6]; PHASE 0: Start
padding 507 pages
aligned heap base=0x0000000000200000 (size=20 MiB)
freemem=27367 ptpages=[L2:3 L1:6 L0:16]; PHASE 1: After parent populates
freemem=27297 ptpages=[L2:13 L1:26 L0:36]; PHASE 2: After parent forks children
freemem=27097 ptpages=[L2:13 L1:26 L0:136]; PHASE 3: After children write
freemem=27097 ptpages=[L2:13 L1:26 L0:136]; PHASE 4: After parent verifies
freemem=27367 ptpages=[L2:3 L1:6 L0:16]; PHASE 5: After all children exit
cowtest: OK
$ ^p
1 sleep  init                  
2 sleep  sh
freemem=32512                  <-- freemem after running cowtest
ptpages=[L2:2, L1:4, L0:4]     <-- ptpages[] after running cowtest

$ QEMU: Terminated
```

### Phase 0 -> 1
  The parent allocates and writes to a 20 MiB heap region. Because the heap is aligned to a 2 MiB boundary, it spans exactly 10 L0 page tables. Populating the heap causes `xv6` to allocate 5120 physical pages for the heap contents and 10 L0 page-table pages to map those pages. As a result, `freemem` decreases from 32497 to 27367, a decrease of 5130 pages. The L0 page-table count increases from 6 to 16, while L1 and L2 remain unchanged.

### Phase 1 -> 2
The parent forks 10 child processes. Each child receives its own root page table and private upper-level page-table structure, but the large heap is not copied. Instead, the heap data pages are shared using copy-on-write, and the heap's L0 page-table pages are also shared between the parent and children.

Even though the heap itself is shared, each child still needs its own trapframe page, root L2 page table, L1 page tables, and L0 page table for trapframe/trampoline mappings. In addition, each child runs briefly after fork and touches its user stack, which causes a copy-on-write stack page and a private copy of the corresponding L0 page table.

Therefore, `freemem` decreases by 70 pages in total, or 7 pages per child. The page-table counters of L2, L1, and L0 are increased by +10, +20, and +20, respectively. The remaining 20 pages are non-page-table pages: 10 trapframe pages and 10 copy-on-write stack pages.


### Phase 2 -> 3
After all children are forked, the parent lets them write to the heap. Each of the 10 children writes to one page in each of the 10 L0 regions, for a total of 100 writes.

Each write triggers two copy-on-write actions. First, the child needs a private copy of the L0 page table for that 2 MiB region, because modifying a leaf PTE inside a shared L0 page table must not affect the parent or the other children. Second, the child needs a private copy of the heap page being written.

Thus, the 100 writes allocate 100 private L0 page-table pages and 100 private heap pages. This decreases `freemem` by 200 pages and increases the L0 page-table count by 100. The L1 and L2 counts do not change because the children already have their own L1 and L2 page tables.

### Phase 3 -> 4
The parent verifies that its heap contents were not modified by the children. This phase only reads from the parent's heap.
Since reads do not trigger copy-on-write, no heap pages are copied. Reads also do not modify page-table entries, so no L0 page-table pages need to be copied. Therefore, `freemem` and all page-table counters remain unchanged.

### Phase 4 -> 5
Finally, all children exit and the parent waits for them. At this point, `xv6` frees each child's private resources: its trapframe, private page-table pages, private stack copy, and private heap pages created by copy-on-write.

Across the 10 children, this releases 270 physical pages. The page-table counters return from `[L2=13 L1=26 L0=136]` to `[L2=3 L1=6 L0=16]`, exactly matching the state after Phase 1. This means all child-private page-table pages were freed, while the parent's original populated heap remains allocated.

The return to the Phase 1 values is important: it shows that the children's COW data pages and private L0 page-table copies were correctly released, and that the parent's shared heap mappings remained intact.

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



