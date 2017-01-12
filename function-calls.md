**main** *(kernel enty point)*

- **kinit1** *(frees pages)*
  
  - **freerange** *(frees pages)*
    
    - **kfree** *(frees single page)*

- **kvmalloc** *(builds new page table)*
  
  - **setupkvm** *(sets up kernel page table)*
  
    - **mappages** *(adds translations to page table)*
    
      - **walkpgdir** *(allocates and maps page)*
      
        - **kalloc** *(allocates page)*
        
        - **memset** *(clean new page)*
  
- **seginit** *(sets up segmentation table)*

- **tvinit** *(initializes interrupt table)*

- **kinit2** *(frees pages)*
  
  - **freerange** *(frees pages)*
    
    - **kfree** *(frees single page)*

- **userinit** *(initialize the First Process)*

  - **allocproc** *(allocates new proc)*
  
  - **setupkvm** *(sets up kernel page table)*
  
    - **mappages** *(adds translations to page table)*
    
      - **walkpgdir** *(allocates and maps page)*
      
        - **kalloc** *(allocates page)*
        
        - **memset** *(clean new page)*
  
  - **inituvm** *(allocates and maps single page, and copies First Process code to it)*

- **mpmain**

  - **idtinit** *(sets %IDTR to point at existing interrupt table)*
  
  - **scheduler** *(runs runnable processes)*
  
    - **acquire** *(locks process table)*
    
      - **pushcli** *(makes us ignore interrupts)*
      
    - **switchuvm** *(prepares proc's kernel stack and makes TSS point to it)*
    
    - **swtch** *(saves current context on proc, and switch to new proc)*
    
    - **switchkvm** *(switches back to kernel page table)*
    
    - **release** *(unlocks process table)*
    
      - **popcli** *(makes us stop ignoring interrupts)*
      
---

**fork** *(creates child process)*

- **allocproc** *(allocates new proc)*

- **copyuvm** *(copies memory)*

  - **setupkvm** *(sets up kernel page table)*
  
    - **mappages** *(adds translations to page table)
    
      - **walkpgdir** *(allocates and maps page)*
      
        - **kalloc** *(allocates page)*
        
        - **memset** *(clean new page)*
  
  - **walkpgdir** *(validate that page mapping exists, without allocating or cleaning)*
  
  - **kalloc** *(allocate new page for user-code)*
  
  - **memmove** *(copy page data)*
  
  - **mappages** *(add user-code page)*
  
    - **walkpgdir** *(maps page, without allocating or cleaning)*
    
  - **freevm** *(free page table in case of error)*
  
    - **deallocuvm** ()
    
      - **walkpgdir** *(get entry of internal page)*
      
      - **kfree** *(free actual page)*
    
    - **kfree** *(free inner table)*
    
    - **kfree** *(free outer table)*
  
- **kfree** *(if error, free kernel-stack)*

- **filedup**

- **idup**

- **safestrcpy**

---
