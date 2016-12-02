###The point of this document:

I'm actually not so sure.
I think the point is to explain why xv6 does stuff, in broad strokes, as a complement to the explanations of the actual code of xv6.

###Important-ish note:

Often, I'll refer to things that haven't been explained yet.
Just keep in mind that not everything is explained at first, and that's okay.

These explanations are not full. They may not even be accurate.
I'm just jotting these down during class, hoping it more or less gives a picture of what's going on.

Anyways, here we go.

---

###*Kernel*
When I say "kernel", I mean the operating system.
This is the mother of all programs that hogs the computer to itself and manages all the other programs and how they access the computer resources (such as memory, registers, time (that is, who runs when), etc.).

The kernel also keeps control over the "mode" bit (in the PS register),
which marks whether current commands are running in user-mode
(and therefore can’t do all kinds of privileged stuff) or kernel-mode (and therefore can do whatever it wants).

###*Boot*
The processor (that is, the actual computer) does not know what operating system is installed, what it looks like, or how large it is.

So how can the processor load xv6 into the memory?

The processor PC register points - by default - to a certain memory place in the ROM, which contains a simple program that:

1.	Copies the very first block (512 bytes. AKA the boot block) from the disc to the memory
2.	Sets the PC to point to the beginning of the newly-copies data

So what?

Every operating system needs to make sure that its first 512 bytes are a small program (it has to be small; it's only 512 bytes!)
that loads the rest of the operating system to the memory and sets the PC to whatever place it needs to be in.
If 512 bytes aren’t enough for this, the program can actually call a slightly larger program that can load and set up more.

In short:

1.	PC points to hard-coded ROM code
2.	ROM code loads beginning of OS code
3.	Beginning of OS code loads the rest of the code
4.	Now hear the word of the Lord!

###*Processes*

A process is the running of a program, including the program’ state and data. The state includes such things as:

-	Memory the program occupied
-	Memory contents
-	Register values
-	Files
- Kernel structures

This is managed by the kernel. The kernel has a simple data structure for each process, organized in some list. The kernel juggles between the processes, using a context switch, which:

1.	Saves state of old process to memory
2.	Loads state of new process from memory

Context switch can be preemptive (i.e. the kernel decides "next guy's turn!", using the scheduler (there'll be a whole lot of talk about this scheduler guy later on)) or non-preemptive (i.e. the hardware itself decides). In non-preemptive, it is asked of programmer to make calls to the kernel once in a while, in order to let the kernel choose to let the next guy run.

Xv6 is preemptive.

Processes are created by the kernel, after another process asks it to. Therefore, the kernel needs to run the first process itself, in order to create someone who will ask for new processes to be created.

###*Fork()*

Every process has a process ID (or pid for short).

A process can call the kernel to do `fork()`, which creates a new process, which is entirely identical to the parent process (registers, memory and everything). The only differences are:

-	The pid
-	The value returned from fork:
 -	In the new process - 0
 -	In the parent process - the new pid
 -	In case of failure - some negative error code

###*Exec()*

`Fork()` creates a new process, and leaves the parent running. `Exec()`, on the other hand, replaces the process's program with a new program. It’s still the same process, but with new code (and variables, stack, etc.). Registers, pid, etc. remain the same.

It is common practice to have the child of `fork` call `exec` after making sure it is the child. So why not just make a single function that does both fork and exec together? There is a brilliant explanation for this, and I don’t remember what it is.

###*Process termination*

Trigger warning: sad stuff ahead. And also zombies.

A process will be terminated if (and only if) one of the following happens:

1.	The process invokes `exit()`
2.	Some other process invokes `kill()` with its pid
3.	The process generates some exception

Note that kill does not actually terminate the process. What it does is leave a mark of "You need to kill yourself" on the process, and it'll be the process itself that commits suicide (after it starts running again, when the scheduler loads it).

Note also that not any process can `kill` any other process. The kernel makes sure of that.

Once `kill`ed, the process's resources (memory, etc.) are not released yet,
until its parent (that is, the process which called for its creation) allows this to happen.
A process that `kill`ed itself but whose parent did not acknowledge this is called a zombie.

In order to a parent to "acknowledge" its child's termination, it needs to call `wait()`.
When it calls `wait()`, it will not continue until one of its children exists, and then it will continue.
If there are a few children, the parent will need to call `wait` once for each child process.

`Wait` returns the pid of the exited process.

What happens if a parent process `exit`s before its children?
Its children become orphans, and the The First Process (whose pid is 1) will make them His children.

###*System calls*

Many commands will only run if the "mode" bit is set to kernel-mode.
However, all processes run on user-mode only; xv6 makes sure of that.

In order to run a privileged command, a process must ask the kernel (which runs in kernel-mode, of course) to carry out the command.
This "asking" is called a system call.

Here’s an example of system call on Linux with x86 processor:

```assembly
movl flags(%esp), %ecx
lea name(%esp), %ebx
movl $5,%eax    ; loads the value 5 into eax register, which is the command "open" in Linux
int $128    ; invokes system call. Always 128!
; eax should now contain success code
```

The kernel has a vector with a bunch of pointers to functions (Yay, pointers!).

###*Addresses*

This one's a biggie. Hold on to your seatbelts, kids.

Programs refer to memory addresses. They do this when they refer to a variable (that's right; once code's compiled, all mentions of the variable are turned into the variable's address). They do this in every `if` or loop. When the good old `EIP` register holds the address of the next instruction, it's referring to a memory address.

There are two issues that arise from this:

* The compiled code does not know where in the memory the program is going to be, and therefore these addresses must be relative to the program's actual address. (This is a problem with loops and ifs, not with the `EIP`.)
* We'll want to make sure no evil program tries to access the memory of another program.

So, each process has to have its "own" addresses, which it thinks are the actual addresses. It has *nothing* to do with the actual RAM, just with the *addresses* that the process knows and refers to. (**Process**, not **program**; this includes the kernel.)

Behold! A sketch of what a process's addresses looks like in xv6:

Address | Who uses this
--- | ---
`[0xFFFF FFFF]` | Kernel
`[0xFFFF FFFE]` | Kernel
... | Kernel
`[0x8000 0000]` | **Kernel**
`[0x7FFF FFFF]` | **Process**
... | Process
`[0x0000 0000]` | Process 

In order to pull off this trick, we use a hardware piece called the Address Translation Unit, which actually isn't officially called that.  
Its real name is the MMU (Memory Management Unit), for some reason.

The MMU is actually comprised of two units:

1. Segmentation Unit

2. Paging Unit.

The MMU sits on the address bus between then CPU and the memory, and decides which actual addresses the CPU accesses when it reads and writes.
Each of the smaller units (segmenataion and paging) can be turned on or off. Note that Paging can only be turned on if Segmentation is on. (We actually won't really use the Segmentation Unit in xv6).

Addresses coming from CPU are called **virtual/logical addresses**. Once through the Segmentation Unit, they're called **linear addresses**. Once through the Paging Unit, they're **physical addresses**.  
In short: CPU [virtual] -> Segmentation [linear] -> Paging [physical] -> RAM

Here's a bunch of 16-bit registers that are used by the Segmentation Unit:

* `CS` - **C**ode. This guy actually messes with our `EIP` register (that's the guy who points to the next command!).
* `DS` - **D**ata. By default, messes with all registers except `EIP`, `ESP` and `EBP`. (In assembly, we can override the choice of messing register.)
* `SS` - **S**tack. By default, messes with `ESP` and `EBP` registers.
* `ES` 
* `FS`
* `GS`

The address `EIP` points to after going through `CS` is written as `CS`:`EIP`.

In the Segmentation Unit there is a register called GDTR. The kernel uses this to store the address of a table called GDT (Global Descriptor Table). Every row is 8 bytes, and row #0 is not used. It can have up to 8192 rows, but no more.

The first two parts of each row are **Base** and **Limit**.

When the Segmentation Unit receives and address, the CPU gives it an *index*. This index is written in one of the segmentation registers (`CS`, `DS`, ... `GS`), thusly:

* Bits 0-1: permissions
* Bit 2: "use GDT or LDT?" (Let's pretend this doesn't exist, because it does not interest us at all.)
* Bits 4-15: The index

The Segmentation Unit receives a logical address and uses the index to look at the GDT. Then:

* If the logical address is greater than the `Limit`, crash!
* Else, the Segmentation adds the `Base` to the logical address, and out comes a linear address.

Note: In the past, the Segmentation Unit was used in order to make sure different processes had their own memory. For example, they could do this by making sure that each time the kernel changes a process, its segmentation regiesters would all point to an index that "belongs" to that process (and each row in the GDT would contain appropriate data). Another way this could be done would be by maintaining a seperate GDT for each process. Or maintaining a single row in the GDT and updating it each time we switch a process. There is no single correct way.

Note that all the addresses used by each process must be consecutive (along the physical memory).

In xv6, we don't want any of this.

Therefore, we will make sure that the GDT `Limit` is set to max possible, and the `Base` is set to 0.

In order to allow consecutive virtual addresses to be mapped to different areas in the physical memory, we use **paging**.

In the Paging Unit, there is a register named `CR3` (Control Register 3), which points to the **physical** address of the Page Table (kinda like GDT). A row in the Page Table has a whole bunch of data, such as page address, "is valid", and some other friendly guys.

When the Paging Unit receives a linear address, it acts thusly:

* The left-side bits are used as an *index* (AKA "page number") for the Page Table. (There are 20 of these.)
* In the matching row, if the "is valid" bit = 0, crash!
* (There are also "permission" bits, but let's ignore them for now.)
* Those "page number" bits from the linear address are replaced by the actual page in our row (which is already *part* of the actual real live physical address)
* The page is "glued" to the right-side bits of the linear address (you know, those that aren't the page number. There are 12 of these.)
* Voila! We have in our hands a physical address.

Note that each page can be in a totally different place in the physical memory. The pages can be scattered (in page-sized chunks) all along the RAM.  
Also note: Hardware demands that the 12 right-most bits of `CR3` be 0. (If not, the hardware'll zero 'em itself.)

**Uh oh**:  
Each row in the Page Table takes up 4KB (that's 12 bits).  
The Page Table has 1024 rows.  
4KB * 1024 = 4MB. That's 4 whole consecutive MBs. That's quite a large area in the memory, which kind of defeats the whole purpose of the Page Table. Well, not the *whole* purpose, but definitely some of it.

**The solution**: The Page Table gets its very own Page Table!

* First (small) page table contains - in each row - the (physical) address of another (small) page table.
* Each of the (small) 2nd-level page tables (which are scattered) contain actual page addresses.
* So: instead of 20 bits for single index, we have 10 bits for 1st-level table index and 10 bits for 2nd-level table index.

Thus, a virtual address can be broken down like so:  
`[i0][i1][offset]`

* `i0` is the index of the First Table
* `i1` is the index of a second table
* `offset` is the offset

###*`1217 main`*

In the beginning, we know that from `[0x0000 0000]` till `[0x0009 FFFF]` there are 640KB RAM.  
From `[0x000A 0000]` till `[0x000F FFFF]` is the "I/O area" (384KB), which contains ROM and stuff we must not use (it belongs to the hardware).  
From `[0x0010 0000]` (1MB) till `[0xFF00 0000]` (4GB - 1MB in total) there is, once again, usable RAM.  
After that comes "I/O area 2".

(Why the 640KB, the break, and then the rest? Because in the olden days they thought no one would ever use more than 640KB.)

Address | Who uses this
--- | ---
`[Who cares]` | I/O
... | I/O
`[0xFF00 0001]` | I/O
`[0xFF00 0000]` | Usable RAM
... | Usable RAM
`[0x0010 0000]` | Usable RAM
`[0x000F FFFF]` | I/O
... | I/O
`[0x000A 0000]` | I/O
`[0x0009 FFFF]` | Usable RAM
... | Usable RAM
`[0x0000 0000]` | Usable RAM

Remember Mr. Boot? He loads xv6 to `[0x0010 0000]`.  
By the time xv6 loads (and starts running `main`), we have the following setup:

1. `ESP` register is pointing to a stack with 4KB, for xv6's use. (That's not a lot.)
2. Segmentation Unit is ready, with a (temporaray) GDT that does no damage (first row inaccessible, and another two with 0 `Base` and max `Limit`.
3. Paging Unit is ready, with a temporary page table. The paging table works thusly:

    * Addresses from `[0x800- ----]` till `[0x803- ----]` are mapped to `[0x000- ----]` through `[0x003- ----]` repectively. (That is, the left-most bit is simply zeroed.)
	* ALL the the above addresses have 1000000000b as their 10 left-most bits, so our 1st-level page table has row 512 (that's 1000000000b) as "valid", and all the rest marked as "not valid".
	* Row 512 points to a single 2nd-level table.
	* The 2nd-level table uses all 1024 of its rows (that's exactly the next 10 bits of our virtual addresses), so they're all marked as valid.
	* Each row in 2nd-level table contains a value which - coincidentally - happens to be the exact same number as the row index.
	
Let's look at some sample virual address just to see how it all works:  
`[0x8024 56AB]` -> `[1000 0000 0010 0100 0101 0110 1010 1011]` -> `[1000 0000 00` (that's row 512) `10 0100 0101` (that's row 581) `0110 1010 1011` (and that's the offset) `]` -> row 581 in the 2nd-level table will turn `[1000 0000 0010 0100 0101...]` to `[0000 0000 0010 0100 0101...]`, which is exactly according to the mapping rule we mentioned a few lines ago.

###*Available pages (free!)*

Processes need pages to be mapped to. Obviously, we want to make sure that we keep track of which page are available.  
The free pages are managed by a **linked list**. This list is held by a global variable named `kmem`. Each item is s `run` struct, which contains only a pointer to the next guy.

`kmem` also has a lock, which makes sure (we'll learn later how) that different processes don't work on the memory in the same time and mess it up. (An alternative would be to give each processor its own pages. However, that could cause some processors to run out of memory while another has spare. (There are ways to work around it.))

`main` calls `kinit1` and `kinit2`, which call `freerange`, which calls `kfree`.  
In `kfree`, we perform the following 3 sanity checks:

* We're not in the middle of some page
* We're not trying to free part of the kernel
* We're not pushing beyond the edge of the physical memory

###*Building a page table*

Let's examine what needs to be done (not necessarily in xv6) in order to make our very own paging table.

Our table should be able to "translate" virtual address *va* to physical address *vp* according to some rule.  
Note that we'll be using `v2p`, which is a hard-coded mother-function that translates **virtual** addresses to **physical** by subtracting `KERNBASE` (which equals `0x8000 0000`) from the virtual addresses.

**Step 1**: Call `kalloc` and get a page for our First Table. (Save (virtual!) address of new table in `pgdir` variable)

**Step 2**: Call `memset` to clear entire page (thus marking all rows as invalid).

**Step 3**: Do the following for **every single _va_** we want to map:

- **Step 3.1**: Create and clear subtable, and save address in `pgtab` (similar to what we did in steps 1 and 2). (SEE NOTE AFTER THIS LIST)  

- **Step 3.2**: Figure out i0 (index in `pgdir`) (using *va* & `v2p` function), write `pgtab` there, mark as valid.

- **Step 3.3**: Figure out i1 (index in `pgtab`) (using *va* & `v2p` function), write *pa* there, mark as valid.

**Step 4**: Set CR3 to point at `pgdir`, using `v2p`.

THE NOTE MENTIONED EARLIER: After we already have some subtables, we only need to create new subtables if the requested subtable does not exist yet.  
How do we know whether it exists already? Simply by looking at the current i0 and seeing whether it's already marked as valid.

ANOTHER NOTE: What if two different *va*->*vp* rules clash? xv6 will crash (deliberately).

**For more details about how the kernel builds its mapping tables, please refer to [xv6 Code Explained.md] (`kvmalloc`).**

###*Moar GDT stuff!*

Remember how we said we'd make sure GDT has `base=0` (so it won't alter addresses) and `limit=max` (so it won't interfere)?  
Well, it turns out the hardware still uses the GDT to check various user-mode/kernel-mode stuff.

We need four rows in the GDT:

1. Kernel-mode, can only execute and read
2. Kernel-mode, can write
3. User-mode, can only execute and write
4. User-mode, can write.

There is a macro named `SEG`, which helps us build all these annoying bits.

Okay, that's enough of this.  
Don't pretend you understand, because you don't and it doesn't matter.

###*Per-CPU variables*

We've got an array of CPU data, `cpus`.

We can access specific SPU stuff via `cpus[SOME_CPU_IDENTIFIER]`.  
In order to get current CPU identifier, we call `getcpu()`, which is slow.

When we do this, we *MUST* stop interrupts from happening, to make sure we stay within the same CPU.

**Problem**: Calling `getcpu()` is slow, and stopping (and then resuming) interrupts can be extremely slow.

**Solution**: Instead of using `getcpu()`, we can use a special register in each CPU!

**Problem**: Registers can be overriden by users.

**Solution**: Special register that only kernel can use!

**Problem**: *There are no such registers.*

**Solution**: Cheating, using the GDT!

So we have a list of GDTs, one per CPU.  
When we initialize a CPU (just once, yeah?), we set up its very own GDT.  
In this GDT, we add a *fifth* row (remember, we have four rows for kernel/user x read/write).  
In this new row, we set the base to point at the place in the memory where the CPU data sits,
and set the limit to 8 (because we have two important variables of CPU data, and each one take 4).

We do this in `seginit()`.

```C
struct cpus *c = &cpus[getcpu()]; // costly, but used just once!

// Set first four rows...
c->gdt[SEG_KCODE]=SEG(STA_X|STA_R, 0, 0xffffffff, 0);
c->gdt[SEG_KDATA]=SEG(STA_S, 0, 0xffffffff, 0);
c->gdt[SEG_UCODE]=SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
c->gdt[SEG_UDATA]=SEG(STA_S, 0, 0xffffffff, DPL_USER);

// Set our special row
c->gdt[SEG_KCPU]=SEG(STA_W, &c->proc, 8, 0);

// Load the GDT
lgdt(c->gdt, sizeof(c->gdt));

// Set GS register to point at our fifth row in the GDT.
loadgs(SEG_KCPU << 3); // shift 3 left so we get value and not permission bits.

// Declare out two variables,
// telling code generator to use gs register
// whenever we access proc or cpu
extern struct proc *proc asm("%gs:0");
extern struct proc *cpu asm("%gs:4");

proc = NULL;
cpu = c;
```

So how does this affect our accessing per-CPU data?  
Normally, if we do `proc = 3;`, then it would be complied to `movl $3, proc`.  
However, now that we did that weird `extern struct proc *proc asm("%gs:0");` part, it is compiled to `movl $3, %gs:0`.

**Therefore**, whenever in the code we call `proc` or `cpu`,
we will automagically be referring to the address held in `GS:0` or `GS:4`,
which happens to be the `proc` or `cpu` belonging to the specific CPU in which the code is executed.

Note that variable `cpu` points to the variables `cpu` and `proc`, so calling `proc` is the same as (yet faster than) calling `cpu->proc`.  
Note also that this means that calling `cpu->cpu->cpu->cpu` is the same as calling `cpu`.

**Important note**: `GS` can be changed by code in user-mode.  
So, every time there is an interrupt, we reload our value to `GS` using `loadgs(SEG_KCPU << 3);`. (Don't worry, we back up all user-mode's registers beforehand.)

###*Processes*

Every process has a struct `proc` that holds a bunch of its data:

* `sz` - memory size
* `pgdir` - its very own page table
* `kstack` - small *kernel* stack (4KB), for data we don't want the process to touch and break (such as return address from system call)
* `state`
* `pid` - process ID
* `parent` - parent process
* `tf` - pointer to place where register values are saved during interrupt
* `context` = another interrupt thing
* `chan` - channel, marks what event process is waiting for (while sleeping). If none, then set to 0.
* `killed` - 0 if not killed yet
* `ofile` - vector of open files
* `cwd` - current working directory (like when doing `cd...` in command)
* `name`

A process can move through its processes thusly:

Embryo -> Runnable -> Running  
Running -> Zombie  
Running -> Sleeping (waiting for whatever event is marked in `chan`)  
Running -> Runnable

The processes are held in a struct named `ptable`, who has a vector of proceses named `proc`.

###*Preparing the First Process*

When running The First Process, we do the following in `userinit`:

* `allocproc`: allocate `proc` structure
 - Allocate process kernel stack
 - Set up data on kernel stack (in the "trapframe" area) *as if the process already existed and went into kernel mode*. As a result, the data will be loaded to proc in due time
* `setupkvm`: create page table (without knowing yet where to map user addresses (only kernel addresses))
* `inituvm`:
 - Allocate free page
 - Copy code to page
 - Update page table to map user memory
* Fix the data on the kernel stack for some reason.

Trapframe contains all register data.

When we return from an interrupt (or, in our case, start the process for the first time), we do the following:

1. Pop the first 4 values to their appropriate registers
2. Pop `eip` field to `eip` register (now `eip` register is pointing to `forkret` function)
3. Goto `forkret` function, which (somehow) takes us to the next guy in the stack...
4. ...Trapret which pops another whole bunch of values from the stack to the registers
5. And so on. We end up with all registers holding good values, and `eip` pointing to 0.

###*The First Process*

All it does it execute the program "init" (with the arguments {"/init", 0} (the 0 is to mark the end of the list)).

This is how the code *would* have looked like if it were written in c:

```c
char *argv[] = {"/init", 0};
main() {
	exec(argv[0], argv);
	for (;;) exit();
}
```

...But it's not written in c.  
It's written in assembly.

Here's the arguments code:

```Assembly
# these are the arguments:

init:
    .string "/init\0"
 
    .p2align 2
argv:
    .long init
    .long 0
```

...And here's the *code* code:

```Assembly
    .globl start
start:
    pushl	$argv ; push second argument
	pushl	$init ; push first argument
	pushl	$0
	movl	$SYS_exec, %eax ; load the canons (perpare to call "exec" system-call)
	int	$T_SYSCALL ; FIRE!
	
# once end up back here, we need to quit.
exit:
	movl	$SYS_exit, %eax
	int	$T_SYSCALL
	jmp	exit
```

So we have this code, but we need to load it during `inituvm`.  
This is done in the script in the **initcode.S** file, which compiles that code to binary and shoves it all into the data section, with the exact same labels that are used in `inituvm`.

###*Actually running the First Process*

This is done in `mpmain`, which calls the scheduler, which loops over process tables and runs processes.

Since at this point there's only one process at this point, it's the only one that'll run.

###*Accessing process's kernel stack during interrupt*

When we're in user-mode and there's an interrupt, the kernel needs its own stack on the process to push real data onto it (before leaving the process).

We need this kernel-stack to be accessable only by the kernel.

The hardware supports this, with a magnificent Rube Goldberg machine:

* `tr` register can only be accessed in kernel-mode
* `tr` contains `SEG_TSS`, which points to special place in GDT
* special place in GDT contains address and size of some weird struct
* weird struct contains field `esp0`, which actually (finally) points to top of process's kernel stack

So during `switchuvm` we set up this whole thing, so that in due time the kernel (and only the kernel) can access its stack on the proc.
