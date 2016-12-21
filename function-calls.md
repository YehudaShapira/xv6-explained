**main** *(kernel enty point)*

- **kinit1** *(frees pages)*

  - freerange (frees pages)
  
    - kfree (frees single page)

- kvmalloc (builds new page table)

  - setupkvm (sets up kernel page table)

- seginit (sets up segmentation table)
- tvinit (initializes interrupt table)
- kinit2 (frees pages)

  - freerange (frees pages)
  
    - kfree (frees single page)

- userinit (initialize the First Process)

  - allocproc (allocates new proc)
  - setupkvm (sets up kernel page table)
  - inituvm (allocates and maps single page, and copies First Process code to it)

-mpmain

  - idtinit (sets %IDTR to point at existing interrupt table)
  - scheduler (runs runnable processes)
  
    - acquire (locks process table)
    
      - pushcli (makes us ignore interrupts)
      
    - switchuvm (prepares proc's kernel stack and makes TSS point to it)
    - swtch (saves current context on proc, and switch to new proc)
    - switchkvm (switches back to kernel page table)
    - release (unlocks process table)
    
      - popcli (makes us stop ignoring interrupts)
