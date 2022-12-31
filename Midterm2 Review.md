# ECE391 Midterm2 Review

## Virtual Memory

###### Lecture slides: 14, 15

### Overview

- Definition

Indirection between memory addresses seen by the software and those used by hardware.

Virtual addresses per program(address space) -- Physical addresses (including memory-mapped IO)

- Advantages: ff

1. [protection] Prevent a malicious program from contaminating or destroying another program’s memory: For a single **program**:
   1.  It seems that virtual memory is sequential, but points to *various locations* in physical memory.
   2. Locations in physical memory that are *not mapped* to the program’s virtual memory are simply *not accessible*.
2. [Effective sharing]
   1. Programs share code in physical memory
   2. code and data not actively used - pushed out to disk - illusion of a much larger physical memory
3. [no fragmentation] When multitask, cannot give contiguous blocks (Paging is better: it doesn't require continuous regions of memory)
4. [simplifies program loading and execution] no relocation of code, rewriting stored pointer values. (paging is better: only need to change a pointer)

- VM Disadvantage:
  - Storage requirement for the paging structure
  - the time overhead to perform translations

- Protection Model

1. Kernel (ring 0), User(ring 3) - lower numbers never call higher; higher calls through system calls
2. CPL(current), RPL(requestor's), DPL(descriptor). -- Restriction: PL = MAX(CPL, RPL) < DPL (**protection fault**)

- 2 levels of indirection: Segmentation and Paging
  1. 32-bit space of physical addresses  - Segment (contiguous portion of a **linear** address space)
  2. Protected mode always use segmentation. 


- Terms

1. **linear address**: an address for any byte in linear address space
2. **physical address**: the processor addresses on its bus, does not have to be contiguous
3. **virtual addresses(paging)**: each program is assigned its own linear address space. Translation is done in hardware by the MMU (hardware)+TLB or operating system (located in MMU)
4. **Logical addresses**(segmentation):  to address a byte in a segment, use a segment selector and an offset

- Virtual memory & physical memory

  - Usually V > P:  **virtualization - ** illusion of multiple unlimited resources
  - P > V: each process can have actual entire memory, avoid moving data in/out of memory, more efficient

- Segmentation terminology

  1. GDTR: 48-bit register points = lower 16 bits (size - 1) + upper 32 bits(location) 
  1. 8B descriptors: # = 0 - 8191 (8 * 8192 = 2^16)
  3. Descriptors: contains information on the starting point, length, and the access rights of the segment
     1. include memory segments, TSS, LDT, Call Gate
     1. #0 is not usable
     1. can differentiate code(executable & possibly readable) & data (readable & possibly writable)

    4. Segment Register: deal with selecting segments of main memory

    5. ```c
       #define KERNEL_CS   0x0010 
       #define KERNEL_DS   0x0018
       #define USER_CS     0x0023 // 0x20+3
       #define USER_DS     0x002B // 0x28+3
       #define KERNEL_TSS  0x0030 // note: these numbers are offsets
       #define KERNEL_LDT  0x0038 // to GDTR, not the segments
       ```

    6. Segment Registers include: Code Segment, Data Segment, Extra data Segment, still more extras Floating S, GS, Stack Segment

    7. Segment Register structure: 15-3 bits mean the index in table, 2 means 0 for GDT, 1 for LDT, 1-0 means RPL

<img src="/Users/Erutsiom/Downloads/IMG_D3F1A14A4CB1-1.jpeg" alt="IMG_D3F1A14A4CB1-1" style="zoom:50%;" />

		7. LDT: per-task segment tables
		7. LDTR: pointer to current LDT (includes base, size index of LDT in GDT)
		7. Segmentation is not used in Linux (base = 0)

### Paging

- Brief explanation

![截屏2022-10-27 16.02.08](/Users/Erutsiom/Library/Application Support/typora-user-images/截屏2022-10-27 16.02.08.png)

A virtual address is translated to a physical address by passing through a **Page Directory** and a **Page Table**.

So there are 2 memory accesses, the first dereferencing the address found at the PD index, indexing into the PT, then dereferencing the address + the 12 bits of offset found in the PT index.

```
pde = read_mem (%CR3 + 4*(la >> 22));
access (pde, user, read);
pte = read_mem ( (pde & 0xfffff000) + 4*((la >> 12) & 0x3ff));
access (pte, user, read);
return (pte & 0xfffff000) + (la & 0xfff);
```

- Bits calculation (1kB = 1024B, B, kB, MB, GB, TB ......)

Page size = 4kB (2^12 B) -> need 12 bits offset

Total size of 32-bit memory = 4GB (2^32 B) -> page # = 2^20

Without hierarchical paging, 2^20 entries(20 bits address), each 32bits -> 2^22 total size(4MB) of PTE

Page the PTE (4kB/page) -> 4kB / 4B= 2^10 entries/page

PTE entries# = 2^20 -> 2^10 PTE tables (2^10 entries for PDE)

PDE entries# = 2^10, also fits in one page

**Relations between size and #:** 

1. For 32-bit memory, length of address = 4B (32 bits)
2. Total_size / length_of_one = # entries
3. If offset = n, it uses (2^n)-byte memory pages

- page states: 1. not exist 2. in physical memory 3. on the disk (1&3: present = 0, differentiate by high 31 bits in PTE structure)
- .align

e.g.: 16位只能从偶数地址取出，32位只能从4的倍数地址取出

```assembly
.align 4
my_array: .long 1000, 400000000, 24, 0 # long means 32-bit value
```

```c
int some_variable __attribute__((aligned (BYTES_TO_ALIGN_TO)));
```

- Trade-off of larger pages
  - a1: save space for page tables
  - a2: more efficient TLB (increase hit rate)
  - a3: faster because reduce memory access (twice for 4kB, once for 4MB)
  - a4: reduce disk I/O (if use disk)
  - d1: Unsuitable allocation leads to memory waste
  - d2: Fragments
  - d3: expensive swap

- segmentation vs paging
  - paging没有外碎片，每个内碎片不超过页的大小。程序不必连续存放。缺点是程序需要全部装入内存
  - segmentation更方便信息共享。对用户可见。大小没有限制。

- codes

```c
#define PDE_TOT_NUM  1024 // total number of pde
#define PTE_TOT_NUM  1024 // total number of pte
#define FourKB           0x1000 // means 4kb
/* Structure for 4Mb page directory entry*/
typedef union pde_4mb_t {
    uint32_t val;
    struct {
        uint32_t present           :1;
        uint32_t read_write        :1;
        uint32_t user_supervisor   :1;
        uint32_t write_through     :1;
        uint32_t cache_disable     :1;
        uint32_t accessed          :1;
        uint32_t dirty             :1;
        uint32_t page_size         :1;
        uint32_t global            :1;
        uint32_t available         :3;
        uint32_t pat               :1;
        uint32_t reserved          :9;
        uint32_t address           :10;
    } __attribute__ ((packed));
} pde_4mb_t;

/* Structure for 4Kb page directory entry*/
typedef union pde_4kb_t {
    uint32_t val;
    struct {
        uint32_t present           :1;
        uint32_t read_write        :1;
        uint32_t user_supervisor   :1;
        uint32_t write_through     :1;
        uint32_t cache_disable     :1;
        uint32_t accessed          :1;
        uint32_t reserved          :1;
        uint32_t page_size         :1;
        uint32_t global_page       :1;
        uint32_t available         :3;
        uint32_t address           :20;
    } __attribute__ ((packed));
} pde_4kb_t;

/* Structure for general page directory entry */
typedef union pde_t {
	pde_4kb_t kb;
	pde_4mb_t mb;
} pde_t;

/* Structure for page table entry*/
typedef union pte_t {
    uint32_t val;
    struct {
        uint32_t present           :1;
        uint32_t read_write        :1;
        uint32_t user_supervisor   :1;
        uint32_t write_through     :1;
        uint32_t cache_disable     :1;
        uint32_t accessed          :1;
        uint32_t dirty             :1;
        uint32_t pat               :1;
        uint32_t global_page       :1;
        uint32_t available         :3;
        uint32_t address           :20; 
    } __attribute__ ((packed));
} pte_t;

extern pde_t page_directory[PDE_TOT_NUM] __attribute__((aligned (FourKB)));
extern pte_t page_table[PTE_TOT_NUM] __attribute__((aligned (FourKB)));
```



### TLB(translatioin lookaside buffers)

- motivation

Way too slow to do on enery memory access.

- features

1. keep translations (first 20 bits)
2. OS manage tables, hardware walks them in x86
3. **flushed** when CR3 is reloaded (e.g. context switch)

- Works with free bits

1. Protect

   1. User/Supervisor: supervisor requires PL < 3
   2. Read_Write

2. Optimize

   1. global: gives same translations for all programs, not flushed when CR3 changes

   2. PS: enable 4MB pages, skip PTE, 22 bits offset - bigger translations (different TLBs)

      4MB pages trade-off:

      ​	a1: makes memory accesses within the kernel very fast

      ​	a2: saves space when storing Page directories

      ​    a3: more efficient TLB
      
      ​	d: bigger fragmentation

#### Codes

```assembly
set_regs:
  pushl %ebp
	movl %esp, %ebp
	movl 8(%esp), %eax
	movl %eax, %cr3 # load dir_pointer in cr3
	movl %cr4, %eax
	orl $0x00000010, %eax # enable 4mb size page in cr4
	movl %eax, %cr4
	movl %cr0, %eax
	orl $0x80000000, %eax # open paging enable bit in cr0
	movl %eax, %cr0
	movl %ebp, %esp
	popl %ebp
	ret
```

```assembly
# x86_desc.S
gdt_desc_ptr:
.word gdt_bottom - gdt - 1
.long gdt

#boot.S
  lgdt    gdt_desc_ptr
```

```assembly
# x86_desc.S
.align  0x1000 # 4096, which is 4kb
page_directory:
_page_directory:
    .rept PDE_TOT_NUM
    .long 0x0
    .endr
page_directory_bottom:

.align  0x1000 # 4096, which is 4kb
page_table:
_page_table:
    .rept PTE_TOT_NUM
    .long 0x0
    .endr
page_table_bottom:
```



## File System

###### Lecture slides: 16

#### Linux File System

[User Space]write() -> [Kernel]system call - Virtual File System-> filesystem -> physical media

- Virtual File System (VFS)

Provides common file system interfaces (create, open, read, write, etc)

- File System Features
  1. a sequence of bytes, tree structure
  2. [everything is considered a file] file types include: regular files(leaves), directories(nodes), symbolic link(point to a file or directory by specifying a path thereto), block-oriented device file, character-oriented device file, pipe (transfer of standard output to some other destination) and named pipes (FIFO) , socket (不同主机的通信 )
  3. Distinction between contents and information
- Terms
  1. Inode: contains information like size, 1-1 map (on the disk)
  2. File descriptor: an index at kernel level of open files - created by progress, represent list of open files. They are passed to read/write/close functions to identify which file they should operate on.
  3. Superblock object: represents a specific mounted FS
  4. inode object: represents a specific file
  5. dentry object: represents a dir entry (entry: glue that holds inodes and files together by relating inode numbers to file names)
  6. file object: represents an open file as associated with a process (**in kernel**, no image on the disk) - each process uses own file object
- File Object impoartant variables
  - *f_op : jump table, file operation structure - generic instance for files on **disk** (different for sockets, device)
  - f_count: 有多少进程打开了它
- Ext2 on disk
  - Boot block
  - Block groups with same structure (1kB to 4kB fixed size)
    - 1kB Super block copy: size, check info (# mount & state), reserved blocks & authentication data, volume name, performance specs
    - 32B group descriptors copy
      - Shortcut to bitmaps, inode table and data blocks
      - Free block count
    - data block bitmap & inode bitmap: free/busy
    - inode table & data blocks
      - Inode_t: i_block[15]: hierarchically
      - <img src="/Users/Erutsiom/Downloads/IMG_38893D7C43AC-1.jpeg" alt="IMG_38893D7C43AC-1" style="zoom:30%;" />
  - disadvantage: 1 too many indirect access, 2 fragment
  - Limitation: hard to insert

#### MP3 File System

<img src="/Users/Erutsiom/Library/Application Support/typora-user-images/截屏2022-10-28 18.02.51.png" alt="截屏2022-10-28 18.02.51" style="zoom:30%;" />

- Max num of (actual) files = 4kB / 4B - 1(statistics) - 1(first dir . ) = 62 (count root, 63)
- Max file size = (4kB - 4Blength) / 4B * 4kB = 4MB - 4kB = 4092kB
- dentry's size - require to be aligned to 64B (40B > 32B), can reduce file name
- limitation on write: does not record free blocks (only consider append operation)
- N and D are initialized during initialization. #dentries will change as adding files
- File type
  - 0 - a file giving user level access to RTC
  - 1 - directory
  - 2 - regular file




## MP devices & methods

### Mode X and Octree

###### MP2.1 & 2.2

- double-buffering
  - switch the display between two screens, drawing only to the screen not currently displayed
  - avoiding the annoying flicker effects associated with showing a partially-drawn image
  - change the registers to 182 x 2 -1?  - Double buffering/scanning
  - not use double buffering for status_bar - Only one place in the memory for status bar
- copy_image - a specific function for VGA copy
  - difference with c lib functions memcpy & strcpy (end for '\b')
- avoid overlap between planes
  - reversing the order (3 2 1 0)
  - a one-byte buffer zone between consecutive planes
  - logical plane is not equal to VGA plane, VGA plane is fixed (0, 1, 2, 3)
- Palette: 64 original + 64 level2 + 128 level4 (selected)
  - can change the palette directly to change the color without redrawing
  - level 2 instead of level 3
    - only 64 left, not enough space for Level 3, level 2 could cover all the pixels
  - level 4 instead of level 3
    - clearer and more accurate pictures (4r4g4b)
- Codes

```c
void show_statusb(int opt, const char* str1, const char* str2){
    int i;
    // ignore the preparation
    /* call help function to convert text to graph */
    text_to_graphics(opt, tmpbar);
    /* finally copy status bar to each plane */
    for(i = 0; i <= 3; i++){ // 4 planes
		SET_WRITE_MASK(1<<(i + 8)); // copy to each plane
		copy_status_bar(statusbar+((3 - i) * STATUS_BAR_SIZE), 0x0000);
	}
}
```

```c
int draw_vert_line (int x){
    unsigned char buf[SCROLL_Y_DIM]; /* buffer for graphical image of line                            */
    unsigned char* addr;             /* address of first pixel in build buffer (without plane offset) */
    int p_off;                       /* offset of plane of first pixel                                */
    int i;                           /* loop index over pixels                                        */
    /* Check whether requested line falls in the logical view window. */
    if (x < 0 || x >= SCROLL_X_DIM) return -1;
    x += show_x; /* Adjust x to the logical colomn value. */
    (*vert_line_fn)(x, show_y, buf); /* Get the image of the line. */
    /* Calculate starting address in build buffer. */
    addr = img3 + (x >> 2) + show_y * SCROLL_X_WIDTH;
    p_off = (3 - (x & 3));  /* Calculate plane offset of first pixel. */
    /* Copy image data into appropriate planes in build buffer. */
    for (i = 0; i < SCROLL_Y_DIM; i++) 
        addr[p_off * SCROLL_SIZE + i * SCROLL_X_WIDTH] = buf[i];
    return 0; /* Return success. */
}
```



### Tux and threads in MP2

###### MP2.2

- threads
  - Synchronization considered in mp2: interrupt (tux thread)
  - Without protection, untimely status (button / LED)
  - Processor(CPU) can only work on one thread at one time
  - Disadvantages of using too many threads
    - extra CPU overhead to handle thread state switching
    - thread safety problems
    - deadlock may occur due to incorrect access order
    - thread will sleep, wake it up will consume a lot of time
- prevent LED spamming (signals are not sent to tux too quickly) - Have a ACK signal
- codes
  - ioctl - 处理user的请求：init / set LED / get button status (copytouser)
  - handle_packet - 处理Tux的消息 

## Process & Thread

###### Lecture slides: 17

- overview
  - one process includes many threads
  - Process - unit of scheduling
  - User-level view
    - Pid - process descriptor / tgid (multithreaded)
  - Kernel view
    - handle many processes simultaneously
    - 8kB structure per process (see filesystem): thread info & kernel stack
    - not use recursion in kernel - quickly override 8kB, panic
  - Task structures
    - cyclic doubly-linked list in kernel
    - Sentinel: init_task created at boot time, persists until shutdown/ reboot
  - System call
    - Translate pid to struct pid* using hash (chaining)
- create processes
  - System calls: fork, fork, clone
  - do_fork in kernel, call copy_process
  - to start a program
    - fork, copy the current program (slow)
    - Exec, loads and starts new program
  - deal with slow fork
    - vfork: parent blocks while child uses address space
    - Copy_on_write: duplicate page tables instead of data
      - when try to write to a page, copy it
      - lazy approach
- Memory map & shared data
  - write - cannot be shared
  - Lib (not write) can be shared by different process
  - Heap - one / process
  - Stack - one / thread
- TSS (task state segment) [picture from slides]
  - Diagram content, interrupt redirection and I/O permission bitmap
  - SS/ESP - user(3), others have number
  - if CPL > IOPL(in EFLAGS), exception generated unless
    - bit map contains enough bits to represent port as a bit
    - bit representing port is set to 0 (has permission)
  - Task switching



## Interrupt

###### Lecture slides: 12, 13

<img src="/Users/Erutsiom/Documents/截屏2022-10-30 11.14.16.png" alt="截屏2022-10-30 11.14.16" style="zoom:30%;" />

- Interrupt better than polling: less waste of resource
- IDT overview
  - 256 entries, each 8bytes
  - must init IDT table before enabling interrupt (lidt, setup_idt())
  - irq_desc: array of descriptors
- Interrupt chaining & soft interrupt (tasklets)
  - Soft routines want to act when interrupt occurs
  - Early use TSR - add a piece of code (cannot remove itself)
  - SA_SHIRQ must be set and match for all 
  - Chaining pros & cons
    - A1: service multiple handlers from a single IRQ line
    - A2: provides scalability while still maintaining simiplicity
    - D1: long interrupts, needs complier action
- control flow
  - [user] receive IRQ
  - Context switch to kernel space (TSS), push user context (IRET)
  - push bookkeeping info
  - [kernel] handler for IRQ based on bookkeeping, do_IRQ() is called which uses the interrupt vector index into the IRQ Desc Table
  - do_IRQ() clears interrupts and works within a critical section
  - teardown, context switch
  - Difference with sys call: also involves context switch, cs runs syscall handler to grant users pribiledge to write to files/ devices or perform other actions

#### functions

- request_irq
  1. sanity check
  2. kmalloc (dynamically allocate struct irqaction) and fill in
     - Flags - IRQF_SHARED(chaining)/DISABLED(execute with IF=0)/SAMPLERANDOM(security)
  3. add new struct to action list - call setup_irq

- setup_irq
  1. sanity check
  2. Random sampling init
  3. **critical section begins (lock+irq)**   check share - chain
     - blocks other activity on descriptor
     - irqsave - allow handlers to be added from any context (e.g. inside an irq)
  4. Not share
     1. sanity check
     2. clear flags
     3. clear depth
     4. startup PIC
  5. **critical section end** create directory, add handler subdirectories
- free_irq: most in critical section, free one action in the list (match with dev_id), remove dir entry
  - Critical section: blocks interrupt from starting to execute (may already be executing on another processor)
  - if it is the last handler, disable irq and invoke PIC shutdown

- do_irq

  1. Get arguments from regs (fastball, save args in EAX, EDX, ECX - reduce memory access)
  2. sanity check
  3. Record regs (per-CPU storage), increments counter of nested handlers
  4. call handle_irq
  5. Exit irq context, process softirqs if needed, set_irq_regs(old) - can be nested

  - processor clears IF when taking interrupt

- handle_level_irq
  1. **lock1** send EOI (mask_ack), preparations and checks
  2. **out of critical section and irq** call handle_IRQ_event 
     - no lock - allow handler use infrastructure
     - IF=1 (changed in handle_IRQ_event without lock) - avoid deadlock
  3. **lock2** remove flag, umask PIC
- Handle_IRQ_event
  1. STI on this CPU unless the first handler has IRQF_DISABLED
  2. Walk through list, call each
  3. CLI

## System call

###### Lecture slides: 19

- Definition: interfaces between User Mode processes and hardware device, need protection boundary (user to kernel mode)
- Structure: (take open as example)

Level 1. Open_wrapper (x86)

  - Save callee-saved regs
  - Move parameters into regs (EDX mode 0x10, ECX flags 0x0C, EBX name 0x08)

 - Move sys_call number into EAX
 - INT 0x80
 - TO LEVEL 2
 - Check invalid return value (call __errno_location)
 - Set return value  EAX (-1 or valid ret value)
 - Restore callee-saved regs
 - RET

Level 2. System_call (x86)

- Save all regs (SAVE_ALL)
- Push parameters to stack
- call *sys_call_table(0, %eax, 4)
- TO LEVEL 3
- Pop parameters
- Put return value in a place (returned in EAX)
- Restore all regs
- Ensure ret value in EAX
- IRET

Level 3. sys_open (C)

- Invokes do_sys_open - LEVEL 4

Level 4. do_sys_open (C)

- get a free file descriptor
- open the file (using do_filp_open - LEVEL 5)
- attach the file to the descriptor

Level 5. filp_open (C)

- check access rights
- find VFS mount
- get a file structure from a free list
- open directory entry - LEVEL 6

Level 6. __dentry_open (C)

- fill in file structure
- call fops->open if valid

#### MP3 System call

- PCB: analogous to a TSS (process descriptor)
  - above the kernel stack
  - <img src="/Users/Erutsiom/Documents/截屏2022-10-30 11.07.19.png" alt="截屏2022-10-30 11.07.19" style="zoom:30%;" />
- File array
  - indexed by file descriptor 
  - is in PCB
  - <img src="/Users/Erutsiom/Documents/截屏2022-10-30 11.08.48.png" alt="截屏2022-10-30 11.08.48" style="zoom:30%;" />

##

## useful things for cheatsheet

```c
void *memcpy(void *dest, const void * src, size_t n)
```



