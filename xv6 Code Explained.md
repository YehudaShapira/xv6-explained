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

**1238**: set up the rest of the pages for general use (because until now we had just minimal)

---

###`2764 struct run`

Represents a memory page.  
A `run` points to the next available page\`run` (which is actually the *previous* page, because the first available is the last in the memory).

---

###`2768 struct kmem`

Points to the head of a list of free (that is, available) pages of memory.

---

###`2780 kinit1(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff).  
Used only when kernel starts up.

---

###`2788 kinit2(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff).  
Used only when kernel starts up.

---

###`2801 freerange(void *vstart, void *vend)`

Frees a bunch of pages.

**2804**: use PGROUNDUP  because `kinit1` in called to start where the kernel finished (which is not likely to end *exactly* at a page end).

---

###`2815 kfree (char *v)`

Frees the (single!) page that `v` points at.  

**2819-2820**: address validity checks

**2823**: fill page with 1s, to help in case of bugs

**2827-2829**: insert our page into the beginning of `kmem` (a linked list with all available pages)

---

###`2838 kalloc(void)`

Removes a page from `kmem`, and returns its (virtual!) address.

**2844-2846**: remove first free page from `kmem`

**2849**: return address

---

###`1757 kvmalloc(void)`

Builds new page table and makes CR3 point to it.

**1759**: make table and get its address

**1760**: make CR3 point to returned address

---

###`1728 kmap[]`

Contains data of how kernel pages should look.  
Used by `setupkvm` for mapping.

Column 0: Virtual addresses  
Column 1: Physical addresses start  
Column 2: Physical addresses end  
Column 3: Pages permissions

**1730-1731**: The `data` variable is where the kernel's data start (*data* is plural, by the way).  
We do not know during compliation where this will be.

---

###`1737 setupkvm(void)`

Sets up kernel virtual pages.

**1742**: create outer `pgdir` page table

**1744**: clear `pgdir`

**1745-1746**: make sure we didn't map illegal address

**1747-1750**: loop over `kmap` and map pages using `mappages`

---

###`1679 mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)`

Creates translations from *`va`* (virtual address) to *`pa`* (physical address) in page table .

**1684**: get starting address

**1685**: get ending address (which is statring address if `size`=1)

**1686**: for each page...

**1687**: get i1 row address (using `walkpgdir`)

**1689**: make sure i1 row not used already

**1691**: write *`pa`* in i1 and mark as valid, with required permissions

---

###`1654 walkpgdir(pde_t *pgdir, const void *va, int alloc)`

Looks at virtual address *`va`*,  
finds where where it should be mapped to according to page table `pgdir`,  
and returns the **virtual** address of the the *index* i1.

If there is no mapping, then:  
if `alloc`=1, mapping is created (and address is returned);  
if `alloc`=0, return 0

Some constants and macros:  
PDX - zeroes offset bits  
PTX - uh... some more bit manipluations  
PTE_P - valid bit  
PTE_W - "can write" bit  
PTE_U - "available in usermode" bit  

**1659**: get i0 index address

**1660**: check if i0 is valid

**1661**: put address of subtable in `pgtab`

**1663**: (i0 not valid) create new subtable, `pgtab`

**1666**: (i0 not valid) clear `pgtab` rows

**1670**: (i0 not valid) point i0 to `pgtab` and mark **i0** as valid, writable, usermodeable.

**1672**: return address of appropriate row in `pgtab`
