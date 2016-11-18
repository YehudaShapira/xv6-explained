Hello, world.

In this document, we’ll attempt to explain the actual code of our beloved xv6.  
Not all of it, but some of the interesting parts.

God have mercy on us.

---

###`1217 main(void)`

The entry point of the kernel.  
Sets up kernel stuff and starts running the first process.

**1219**: set up first bunch of pages, for kernel to work with (minimal, because old harware has little memory)

**1238**: set up the rest of the pages (because until now we had just minimal)


---

###`2780 kinit1(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff)  
Used only when kernel starts up.

---

###`2788 kinit2(void *vstart, void *vend)`

Frees a bunch of pages.  
Also does some locking thing (I'll elaborate once we actually learn this stuff)  
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
