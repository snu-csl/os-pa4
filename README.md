# 4190.307 Operating Systems (Spring 2024)
# Project #4: KSM (Kernel Samepage Merging)
### Due: 11:59 PM, May 26 (Sunday)

## Introduction

KSM (Kernel Samepage Merging) is a memory de-duplication feature that enables the kernel to consolidate identical memory pages into a single shared page across multiple processes, thereby saving memory. The objective of this project is to explore the virtual memory subsystem of `xv6` by implementing the KSM feature. 

## Background: Linux KSM

Linux Kernel Samepage Merging (KSM) is a memory-saving de-duplication feature that allows the Linux kernel to examine and combine identical memory pages into a single page that is shared among multiple processes. This process is managed through a technique called copy-on-write, where any attempt to modify the shared page results in the creation of a new page for the modifying process, thus preserving the integrity of the original shared page. KSM is particularly useful in virtualized environments where multiple instances might run similar sets of data, allowing more efficient use of memory resources. For additional details about Linux KSM, you can visit [this page](https://docs.kernel.org/admin-guide/mm/ksm.html). 

## Problem specification

### 1. Implement the `ksm()` system call (70 points)

First, implement the `ksm()` system call. The system call number of `ksm()` is already assigned to 24 in the `./kernel/syscall.h` file. 

__SYNOPSYS__
```
    int ksm(int *scanned, int *merged);
```

__DESCRIPTION__

The `ksm()` system call performs kernel samepage merging (KSM) for `xv6`. It is designed to be periodically invoked by a user-level process called `ksmd`. It scans all the physical page frames used by user processes to identify a set of page frames with identical contents. It then merges these frames, keeping only one physical copy in memory. 

__RETURN VALUE__

* The `ksm()` system call returns the number of free page frames (in 4KiB) after performing KSM.
* `ksm()` also records the total number of scanned frames and merged frames into the locations pointed to by the `scanned` and `merged` arguments, respectively.

The overall flow for performing KSM can be outlined as follows:
1. Scan the page frames used by each user process `p` from the virtual address 0 to `p->sz`. The region will contain code, data, heap, and stack (+ stack guard) pages. Exclude trampoline, trapframe, kernel stack, and page table pages from scanning as they are not targets for KSM. Note that Linux KSM merges only anonymous pages because Linux supports demand paging and shared `mmap()` for file-backed pages. In `xv6`, however, all the code and data pages are initially copied from an executable file to physical memory during `exec()`. As a result, when multiple processes are created from the same executable file, each process will have its own private copies of code and data pages.
   
2. Performing a byte-to-byte comparison on two page frames to determine if their contents are identical is time-consuming. Instead, use hashing to identify page frames with identical contents. A hash function called `xxh64()` is provided in the skeleton code (@ `kernel/xxh.c`). This function is a port of the [`xxh` hashing algorithm](https://xxhash.com/) used in Linux KSM, tailored for 64-bit little-endian RISC-V CPUs. Relying solely on hash values to identify identical page frames can be risky due to potential hash collisions. However, the probability of hash collisions is quite low, so you don't need to worry about them for this project assignment.

3. Once you find an existing page frame with the same content as the current one, update the corresponding page table entry (PTE) to point to the existing page frame and release the current page frame. Please note that when a page frame is shared by multiple virtual pages, page write permissions (`W` bit in PTE) should be removed from all the PTEs referencing the shared page frame.

4. You are required to preallocate a "___zero-page___" in the system which is filled with zeroes. Any page frames that are entirely zero-filled, such as BSS and heap pages initialized to zeroes, can be mapped to this zero-page.  

5. If a process attempts to write data to a shared page frame, it will trigger a page fault. In this situation, you should perform copy-on-write (COW) by allocating a new page frame, copying the content from the shared page frame, and updating the corresponding PTE to reference the new private page frame while enabling the write permission. Later, this private page frame can be merged again if another page frame with identical content is found.

6. When a process terminates, only the private page frames should be released.

### 2. Design document (30 points)

You need to prepare and submit the design document (in a single PDF file) for your implementation. Your design document should include the following:

1. New data structures
  * Describe any new data structures introduced for this project.
  * Explain the purpose and functionality of each data structure.
  * Provide insights into why these data structures were necessary and how they contribute to the KSM implementation.

2. Overall flowchart 
  * Include a flowchart illustrating the overall process of KSM.
  * Explain each step of the flowchart and its purpose in the KSM implementation.
  * Highlight the key decisions and branching points within the flow.

3. Algorithm design
  * Describe the algorithm used for scanning and merging page frames.
  * Provide pseudocode or a detailed explanation of the algorithm.
  * Explain any corner cases you have considered and the strategies you used to address them.
  * Discuss any efforts you made to enhance the code efficiency, both in terms of time and space.

4. Implementation details
  * Provide an overview of your code structures.
  * Highlight important functions and their roles.

5. Testing and validation
  * Describe the testing strategy used to verify the KSM implementation
  * Discuss how you verify the correct handling of the corner cases highlighted in Section 3. 

## Restrictions

* For this project assignment, you can assume a uniprocessor RISC-V system (`CPUS` = 1) with a physical memory size of 128 MiB.
* You can assume that a single page frame can be shared by up to 16 different virtual pages, except for the zero-page. There is no limit on the number of virtual pages that can share the zero-page.
* When performing `ksm()`, exclude the page frames used by `init` process (pid 1), the `sh` process (pid 2), and the process invoking the `ksm()` system call itself from scanning. Assume that the shell process always runs with pid 2.
* Duplicated page frames are merged only through the `ksm()` system call; they should not be merged at the time of page allocation.
* There should be no memory leak. The `freemem` value should remain identical before and after executing a program. 
* Please use the `qemu` version 8.2.0 or later. To determine the `qemu` version, use the command: `$ qemu-system-riscv64 --version`
* We will run `qemu-system-riscv64` with the `-icount shift=0` option, which enables aligning the host and virtual clocks. This setting is already included in the `Makefile` for the `pa4` branch.
* You only need to change the files in the `./kernel` directory (mostly to `ksm.h` and `ksm.c` files provided in the skeleton code). Any other changes outside the `./kernel` directory will be ignored during grading.

## Skeleton code

The skeleton code for this project assignment (PA4) is available as a branch named `pa4`. Therefore, you should work on the `pa4` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa4
```

After downloading, you must first set your `STUDENTID` in the `Makefile` again.
The `pa4` branch includes a ksm daemon program called `ksmd`, and three user-level programs called `ksm1`, `ksm2`, and `ksm3`, whose source code is available in `./user/ksmd.c`, `./user/ksm1.c`, `./user/ksm2.c`, and `./user/ksm3.c`, respectively.

The `ksmd` program is responsible for invoking the `ksm()` system call at regular intervals, typically every 5 ticks (Note: this interval can be adjusted during actual tests.). It then prints the number of scanned and merged page frames, as well as the `freemem` information. The skeleton code already tracks the `freemem` value by incrementing or decrementing it by one for every `kalloc()` or `kfree()` function call within the kernel. Therefore, you can directly use this value as the return value of the `ksm()` system call. 

The user-level programs, `ksm1`, `ksm2`, and `ksm3`, serve as test programs for KSM. Specifically, the `ksm1` program sequentially forks three child processes and then terminates them one by one in a different order. During the `fork()` operation, the content of the physical memory is copied to each child process. The kernel should be able to identify and merge these duplicated contents through the `ksm()` system call.

The `ksm2` program is designed to test the correct handling of writes to shared page frames. Those shared page frames should be set to read-only, and any write attempt should trigger a page fault. The page fault handler is then responsible for performing COW (copy-on-write) operation, creating a private copy of the frame for the process that caused the fault.

Finally, the `ksm3` program conducts a stress test by continuously updating each element in a large array, spanning 16 pages. As these updates occur, the `ksmd` process attempts to find and merge any page frames with identical content. However, due to ongoing updates, these merged frames will frequently undergo a COW operation again. At the end of the program, all elements in the array should have the same value. Otherwise, it indicates you didn't correctly handle the updates to the shared page frames. 

The `ksm3` program is intended to be executed with the `ksmd` process running in the background, as illustrated in the following example (Note: the initial `freemem` value can vary depending on your implementation.):

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -icount shift=0 -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

init: starting sh
$ ksmd &
ksmd: scanned 0, merged 0, freemem 32564
$ ksm3
...
```

## Tips

* Read Chap. 3 and 4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-1/book-riscv-rev3.pdf) to understand the virtual memory subsystem and page-fault exceptions in xv6.

* For your reference, the following roughly shows the amount of changes you need to make for this project assignment. Each `+` symbol indicates 1~10 lines of code that should be added, deleted, or altered.
```
 kernel/defs.h    | +
 kernel/exec.c    | +
 kernel/proc.c    | +
 kernel/trap.c    | +
 kernel/vm.c      | +
 kernel/ksm.h     | +++++
 kernel/ksm.c     | ++++++++++++++++++++++++++++++++++++++++
```
  
## Hand in instructions

* First, make sure you are on the `pa4` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file with your design document (in PDF format) to the submission server.
  
* The total number of submissions for this project assignment will be limited to 50. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, 40 at hours between 09:00 and 00:40 the following day, and at every hour (minute 0) between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time (time it can take to run the script), send the request again, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, that is likely due to an error in the automated process. The TAs will check if the script is properly running, but that is a ___manual___ process, so please do not expect it to be completed immediately.


## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use in the QnA board of the submission server right after each submission. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
