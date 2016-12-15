Hello, world.

In this document, we’ll attempt to explain the actual code of our beloved xv6.  
Not all of it, but some of the interesting parts.

God have mercy on us.

---

###`1217 main(void)`

The entry point of the kernel.  
Sets up kernel stuff and starts running the first process.

**1219**: set up first bunch of pages, for kernel to work with (minimal, because old harware has little memory)

**1220**: set up all kernel pages

**1223**: set up segment tables and per-CPU data

**1238**: set up the rest of the pages for general use (because until now we had just minimal, because other CPUs might not handle high addresses)

**1239**: set up the First Process

**1241**: run the scheduler (and the First Process)

---

###`2764 struct run`

Represents a memory page.  
A `run` points to the next available page/`run` (which is actually the *previous* page, because the first available is the last in the memory).

---

###`2768 struct kmem`

Points to the head of a list of free (that is, available) pages of memory.

---

###`2780 kinit1(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff).  
Used only when kernel starts up.

Called by [`main`](#1217-mainvoid).

---

###`2788 kinit2(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff).  
Used only when kernel starts up.

Called by [`main`](#1217-mainvoid).

---

###`2801 freerange(void *vstart, void *vend)`

Frees a bunch of pages.

**2804**: use PGROUNDUP  because `kinit1` in called to start where the kernel finished (which is not likely to end *exactly* at a page end).

Called by:

* [`kinit1`](#2780-kinit1void-vstart-void-vend)

* [`kinit2`](#2788-kinit2void-vstart-void-vend)

---

###`2815 kfree(char *v)`

Frees the (single!) page that `v` points at.  

**2819-2820**: address validity checks

**2823**: fill page with 1s, to help in case of bugs

**2827-2829**: insert our page into the beginning of `kmem` (a linked list with all available pages)

Called by:

* `deallocuvm`

* `freevm`

* `fork`

* `wait`

* [`freerange`](#2801-freerangevoid-vstart-void-vend)

* `pipealloc`

* `pipeclose`

---

###`2838 kalloc(void)`

Removes a page from `kmem`, and returns its (virtual!) address.

**2844-2846**: remove first free page from `kmem`

**2849**: return address

Called by:

* `startothers`

* [`walkpgdir`](#1654-walkpgdirpde_t-pgdir-const-void-va-int-alloc)

* [`setupkvm`](#1737-setupkvmvoid)

* [`inituvm`](#1803-inituvmpde_t-pgdir-char-init-uint-sz)

* `allocuvm`

* `copyuvm`

* [`allocproc`](#2205-allocprocvoid)

* `pipealloc`

---

###`1757 kvmalloc(void)`

Builds new page table and makes `%CR3` point to it.

**1759**: make table and get its address

**1760**: make `%CR3` point to returned address

Called by [`main`](#1217-mainvoid).

---

###`1728 kmap[]`

Contains data of how kernel pages should look.  
Used by `setupkvm` for mapping.

**Column 0**: Virtual addresses  
**Column 1**: Physical addresses start  
**Column 2**: Physical addresses end  
**Column 3**: Pages permissions

**Note**: The `data` variable (used in lines 1730-1731 is where the kernel's data start (*data* is plural, by the way).  
We do not know during compliation where this will be.

---

###`1737 setupkvm(void)`

Sets up kernel virtual pages.  
**Returns** page table address if successful, 0 if not.

**1742**: create outer `pgdir` page table

**1744**: clear `pgdir`

**1745-1746**: make sure we didn't map illegal address

**1747-1750**: loop over `kmap` and map pages using `mappages`

Called by:

* [`kvmalloc`](#1757-kvmallocvoid)

* `copyuvm`

* [`userinit`](#2252-userinitvoid)

* `exec`

---

###`1679 mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)`

Creates translations from *`va`* (virtual address) to *`pa`* (physical address) in existing page table `pgdir`.  
**Returns** 0 if successful, -1 if not.

**1684**: get starting address

**1685**: get ending address (which is starting address if `size`=1)

**1686**: for each page...

**1687**: get i1 row address (using `walkpgdir`)

**1689**: make sure i1 row not used already

**1691**: write *`pa`* in i1 and mark as valid, with required permissions

Called by:

* [`setupkvm`](#1737-setupkvmvoid)

* [`inituvm`](#1803-inituvmpde_t-pgdir-char-init-uint-sz)

* `allocuvm`

* `copyuvm`

---

###`1654 walkpgdir(pde_t *pgdir, const void *va, int alloc)`

Looks at virtual address *`va`*,  
finds where where it should be mapped to according to page table `pgdir`,  
and returns the **virtual** address of the the *index* i1.  
**Returns** address if successful, 0 if not.

If there is no mapping, then:  
if `alloc`=1, mapping is created (and address is returned);  
if `alloc`=0, return 0

Some constants and macros used here:  
PDX - zeroes offset bits  
PTX - uh... some more bit manipluations  
PTE_P - "valid" bit  
PTE_W - "can write" bit  
PTE_U - "available in usermode" bit  

**1659**: get i0 index address

**1660**: check if i0 is valid

**1661**: put address of subtable in `pgtab`

**1663**: (i0 not valid) create new subtable, `pgtab`

**1666**: (i0 not valid) clear `pgtab` rows

**1670**: (i0 not valid) point i0 to `pgtab` and mark **i0** as valid, writable, usermodeable.

**1672**: return address of appropriate row in `pgtab`

Called by:

* [`mappages`](#1679-mappagespde_t-pgdir-void-va-uint-size-uint-pa-int-perm)

* `loaduvm`

* `deallocuvm`

* `clearpteu`

* `copyuvm`

* `uva2ka`

---

###`1616 seginit(void)`

Sets segmentation table so that it doesn't get in the way, for each CPU. 
Adds extra row to each segementation table in order to guard CPU-specific data, makes `%gs` resgister point to it, and makes `proc` and `cpu` actually point to `%gs`.

**1624-1628**: set up regular rows in segmentation table

**1631-1634**: set up special row and `%gs` register

**1637-1638**: set up inital `proc` and `cpu` data

Called by [`main`](#1217-mainvoid).

---

###`2252 userinit(void)`

Creates and sets up The First Process.

**2255**: data and size of First Process code. Filled by script during compilation.

**2257**: allocate `proc` structure and set up data on kernel stack

**2258**: save proc in `initproc`, so we'll always remember who the First Process is

**2259-2260**: create page table with kernel addresses mapped

**2261**: allocate free page, copy process code to page, map user addresses in page table

**2262-2275**: fix data on kernel-stack, *as if* it were stored there because of an interrupt

- **2264**: make sure we'll be in usermode when process starts

- **2270**: make sure process will start in address 0

Called by [`main`](#1217-mainvoid).

---

###`2205 allocproc(void)`

Allocates `proc` structure and sets up data on kernel stack.  
**Returns** proc if succeeds, 0 if not.

**2211-2213**: find first unused `proc` structure

**2217-2219**: set to EMBRYO state and give `pid`

**2223-2226**: allocate and assign kernel stack for process

**2227**: set stack pointer to bottom of stack (stack bottom is highest address in stack)

**2230-2231**: make room on stack for `trapframe`

**2235-2239**: make room on stack for `context`

**2240-2241**: set `context`, setting `context.eip` to function `forkret`

Called by:

* [`userinit`](#2252-userinitvoid)

* `fork`

---

###`1803 inituvm(pde_t *pgdir, char *init, uint sz)`

Allocates and maps single page (4KB), and fills it with with program code.

**1807-1808**: make sure entire program code fits in single page

**1809**: allocate page

**1810**: clear page

**1811**: map pages in page table `pgdir` (using `v2p`, because we know what memory this is, because this is the First Process, which the kernel always creates on startup)

**1812**: copy the code

Called by [`userinit`](#2252-userinitvoid).

---

###`2458 scheduler(void)`

Loops over all processes (in each CPU), finds a runnable process, and runs it.  
Loops for ever and ever.

**2467**: lock process table, to prevent multiple CPUs from grabbing same process

**2468-2470**: find first available process

**2475**: set *per-CPU* variable `proc` to point to current running process

**2476**: set up process's kernel stack, and switch to its page table

**2477**: mark process as running

**2478**: save current registers (including where to continue on scheduler) and load process's registers, handing the stage over to the process  
(NOTE: the running process is responsible to release the process table lock (to enable interrupts) and later release it.)

**2479**: now that process is switching back to scheduler, switch back to kernel registers and Page Table

**2483**: set per-CPU variable `proc` back to 0

**2485**: release process table, allowing other CPUs to grab processes (just in case 

Called by `mpmain`.

---

###`1773 switchuvm(struct proc *p)`

Perpares kernel-stack of process (that is, makes `tr` register indirectly point to it), and loads process's Page Table to `cr3`.

**1776-1779**: set up `%tr` register and `SEG_TSS` section in GDT end up magically (don't ask how) referring us to top of process's kernel stack

Called by:

* `growproc`

* [`scheduler`](#2458-schedulervoid)

* `exec`

---

###`2708 swtch(struct context **old, struct context *new)`

Saves current register context in `old`, then loads the register context from `new`.  
Basically gives control to new process.

**2709**: set `%eax` to contain address of `old` context

**2710**: set `%edx` to contain `new` context

**2713-2716**: push `%ebp`, `%ebx`, `%esi`, `%edi` onto current stack (which happens to be `old` stack)

**2719**: copy value of `%esp` to address held in `%eax`, which is the `old` stack address (see line **2709**)

**2720**: set current stack pointer (`%esp`) to value of `%edx`, which is the `new` stack address (see line **2710**)

**2723-2726**: pop `%edi`, `%esi`, `%ebx`, `%ebp` from `new` stack onto the actual stack

Called by:

* [`scheduler`](#2458-schedulervoid)

* [`sched`](#2504-schedvoid)

---

###`2503 sched(void)`

Switches back `scheduler` to return from a process that had enough running.

Called by:

* `exit`

* [`yield`](#2522-yieldvoid)

* `sleep`

---

###`2522 yield(void)`

Gives up the CPU from a running process.

**2524**: re-lock the process table for scheduler

**2525**: make self as not running

**2526**: switch back to scheduler

**2527**: after scheduler re-ran process, re-ealease process table to enable interrupts

Called by `trap`.

---

###`2553 sleep(void *chan, struct spinlock *lk)`

Makes process sleep until `chan` event occurs.

**2568**: lock process table in order to set sleeping state safely

**2569**: now that process table is locked, release `lk`

**2573-2574**: set up sleeping state (and alarm clock)

**2575**: return to scheduler until the event manager marks process as runnable

**2578**: clean up

**2582**: release process table

**2583**: lock `lk` once again

---

###`1555 pushcli(void)`

Saves state of `%eflags` register's `IF` bit (that is, the current state of "listen to interrupts?" bit),  
increments the "how many times did we choose to ignore interrupts" counter,  
and clears the "listen to interrups" bit.

**1561-1562**: save initial state of bit in `interna` var (FL_IF is the location of our bit)

Called by:

* [`acquire`](#1474-acquirestuct-spinlock-lk)

* [`switchuvm`](#1773-switchuvmstruct-proc-p)

---

###`1566 popcli(void)`

Decrements the "how many times did we choose to ignore interrupts" counter,  
and if it reaches 0 then sets the "listen to interrups" bit to whatever it was before the very first `pushcli` was ever called.

**1572-1573**: only set our bit if `interna` (initial bit value) was set

Called by:

* [`release`](#1502-releasestruct-spinlock-lk)

* [`switchuvm`](#1773-switchuvmstruct-proc-p)

---

###`1474 acquire(struct spinlock *lk)`

Loops over spinlock until lock is acquired (exclusively by current CPU).

**1476**: disable interrupts (which will be enabled in `release`)

**1483-1484**: loop over lock until it has a zero (and write "1" in it)

---

###`1502 release(struct spinlock *lk)`

Releases spinlock from being held by current CPU.

**1519**: write "0" in lock

**1521**: re-enable interrups (which were disabled in `acquire`)

---

###`3004 alltraps`

Catches and prepares all interrupts for `trap`.  
Pushes register data on stack, calls `trap` with the `stack` as a `trapframe` argument, pops register data from stack, and finally calls `iret`.

**3005-3010**: store registers and build `trapframe`

**3013-3018**: set up data and per-CPU segments (?)

**3021-3023**: call `trap`, using stack as argument

**3027-3034**: pop registers and call `iret`

---

###`3101 trap(struct trapframe *tf)`

Handles all interrups.

**3106**: save `tf` to `proc->tf`, so that we don't need to start passing it around to `syscall`

Called by [`alltraps](#3004-alltraps)

---

###`3067 tvinit(void)`

Initializes the IDT table.

Called by [`main`](#1217-mainvoid).

---

###`3079 idtinit(void)`

Makes `%IDTR` point at existing IDT table.

Called by `mpmain`

---
