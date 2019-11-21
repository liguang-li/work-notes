# Compound pages
A compound page is simply **a grouping of two or more physical contiguous pages** into a unit that can, in many ways, be treated as a **single, large page**. They are most commonly used to create huge pages, used within __hugetlbfs__ or the  __transparant huge page__ subsystem, but they show up in other contexts as well.
Compound pages can serve as anonymous memory or be used as buffers within the kernel, they cannot. however, appear in the page cache, which is only prepared to deal with singleton page.

Allocating a compound pages is a matter of calling a normal memory allocation function like *alloc_pages()* with the \_\_GFP\_COMP allocation flag set and an order of at least one. (The "order" of an allocation is the base-2 logarithm of the number of pages to allocate; zero thus corresponds to a single page, one to two pages, etc.). It is not possible to create an order-zero (single-page) compound page due to the way compound pages are implemented.

Note that a compound page differs from the pages returned from a normal higher-order allocation request. A call like

~~~c
pages = alloc_pages(GFP_KERNEL, 2);  /* no __GFP_COMP */
~~~
will return four physically contiguous pages, but they will not be a compound page. The difference is that creating a compound page involves the **creation of a fair amount of metadata**; much of the time, that metadata is unneeded so the expense of creating it can be avoided.
## Metadata
what does that metadata look like?
Most of it is stored in the associated page structures.

* 64-bit systems
  The first (normal) page in a compound page is called the "head page"; it has the PG_head flag set. All other pages are "tail pages"; they are marked with PG_tail.

* 32-bit systems
  No page flags to spare, so a different scheme is used.
  All pages in a compound page have the PG_compound flag set, and the tail pages have PG_reclaim set as well.
  The PG_reclaim bit is normally used by the page cache code, but, since compound pages cannot be represented in the page cache, that flag can be reused here.

*PageCompound()* : return a true value if the passed-in page is a compound page.
*PageHead()* :
*PageTail()* :

Every tail page has a pointer to the head page stored in the *first_page* field of *struct page*. This field occupies the same storage as the *private* field, the spinlock used when the page holds page table entries, or the slab_cache pointer used when the page is owned by a slab allocator.

The *compound_head()* helper function can be used to find the head page associated with any tail page.

The *order* is stored in the *lru.prev* field in the *page* structure for the **first tail page**.

While unions are used for many of the overlaid fields in struct page, here the order is simply cast into a pointer type before being stored in a pointer field. Similarly, a pointer to the destructor is stored in the *lru.next* field of the **first tail page's struct page**.

This extension of compound-page metadata into the second page structure is why compound pages must consist of at least two pages.

There are only two compound page destructors declared in the kernel. By default, *free_compound_page()* is used; all it does is return the memory to the page allocator.

The hugetlbfs subsystem, though, uses *free_huge_page()* to keep its accounting up to date.

In most cases, compound pages are unnecessary and ordinary allocations can be used; calling code needs to remember how many pages it allocated, but otherwise the metadata that would be stored in a compound page is unneeded.

A compound page is indicated, though, whenever it is important to **treat the group of pages as a whole even if somebody references a single page within it**.

Transparent huge pages are a classic example; if user space attempts to change the protections on a portion of a huge page, the entire huge page will need to be located and broken up. Various drivers also use compound pages to ease the management of larger buffers.

~~~c
void prep_compound_page(struct page *page, unsigned int order)
{
        int i;
        int nr_pages = 1 << order;

        set_compound_page_dtor(page, COMPOUND_PAGE_DTOR);
        set_compound_order(page, order);
        __SetPageHead(page);  //PG_head
        for (i = 1; i < nr_pages; i++) {
                struct page *p = page + i;
                set_page_count(p, 0);
                p->mapping = TAIL_MAPPING;
                set_compound_head(p, page);
        }
        atomic_set(compound_mapcount_ptr(page), -1);
}

static inline void set_compound_page_dtor(struct page *page,
                enum compound_dtor_id compound_dtor)
{
        VM_BUG_ON_PAGE(compound_dtor >= NR_COMPOUND_DTORS, page);
        page[1].compound_dtor = compound_dtor;
}

static inline void set_compound_order(struct page *page, unsigned int order)
{
        page[1].compound_order = order;
}

static __always_inline void set_compound_head(struct page *page, struct page *head)
{
        WRITE_ONCE(page->compound_head, (unsigned long)head + 1); 
}

static inline struct page *compound_head(struct page *page)
{
        unsigned long head = READ_ONCE(page->compound_head);
		/*Page belongs to compound pages, except the PG_head page*/
        if (unlikely(head & 1))
                return (struct page *) (head - 1);
    	/*Normal page or PG_head page*/
        return page;
}

static __always_inline int PageTail(struct page *page)
{
        return READ_ONCE(page->compound_head) & 1;
}

static __always_inline int PageCompound(struct page *page)
{
        return test_bit(PG_head, &page->flags) || PageTail(page);
}

static inline unsigned int compound_order(struct page *page)
{
        if (!PageHead(page))
                return 0;
        return page[1].compound_order;
}
~~~
