+++
title = 'Wiredtiger - Cache相关概念梳理'
date = 2024-11-29
tags = ["Wiredtiger", "Cache", "存储引擎", "数据库"]
draft = true
+++

## Concept on Disk
### Block Manager:
[https://source.wiredtiger.com/11.3.1/arch-block.html](https://source.wiredtiger.com/11.3.1/arch-block.html)

+ A block is a chunk of data that is stored on the disk and operated on as a single unit.
+ <font style="color:rgb(55, 55, 55);">Each WiredTiger data file (any file in the home directory with the .wt suffix) is made up of these blocks. </font>
+ <font style="color:rgb(55, 55, 55);">Each block consists of a page header, a block header and contains a single page of the btree from which it was generated.</font>
+ <font style="color:rgb(55, 55, 55);">WiredTiger is a no-overwrite storage engine, and when blocks are re-written, they are written to new locations in the file. </font>

![](https://cdn.nlark.com/yuque/0/2024/png/47415822/1732412940642-407465b0-8098-42a0-ad30-b181162248d3.png)

### Descriptor blocks:
+ A file is divided up into blocks. The first block in a file is special as it contains metadata about the file and is referred to as the "descriptor block". It contains the WiredTiger major and minor version, a checksum of the block contents as well as a "magic" number to check against.

### Data File Format:
[https://source.wiredtiger.com/11.3.1/arch-data-file.html](https://source.wiredtiger.com/11.3.1/arch-data-file.html)

<font style="color:rgb(55, 55, 55);">Btree <----> Table 1 : 1</font>

+ <font style="color:rgb(55, 55, 55);">In memory, a database table is represented by a B-Tree data structure, which is made up of nodes that are page structures. </font>
+ <font style="color:rgb(55, 55, 55);">The root and internal pages will only store keys and references to other pages, while leaf pages store keys, values and sometimes references to overflow pages. </font>

<font style="color:rgb(55, 55, 55);">Block <----> Page  1 : 1</font>

+ <font style="color:rgb(55, 55, 55);">When these pages are written onto the disk, they are written out as units of data called blocks. </font>
+ <font style="color:rgb(55, 55, 55);">On the disk, a WiredTiger data file is just a collection of blocks which logically represent the pages of the B-Tree.</font>

Data File --- Block --- Page on Disk

+ <font style="color:rgb(55, 55, 55);">Each WiredTiger data file (any file in the home directory with the .wt suffix) is made up of these blocks. </font>
+ <font style="color:rgb(55, 55, 55);">On the disk, a WiredTiger data file is just a collection of blocks which logically represent the pages of the B-Tree.</font>
+ <font style="color:rgb(55, 55, 55);">The layout of a .wt file consists of a file description </font>`<font style="color:rgb(55, 55, 55);">WT_BLOCK_DESC</font>`<font style="color:rgb(55, 55, 55);"> which always occupies the first block, followed by a set of on-disk pages. </font>

### <font style="color:rgb(55, 55, 55);">Page on Disk</font>
+ <font style="color:rgb(55, 55, 55);">Pages consist of a header (</font>`<font style="color:rgb(55, 55, 55);">WT_PAGE_HEADER</font>`<font style="color:rgb(55, 55, 55);"> and </font>`<font style="color:rgb(55, 55, 55);">WT_BLOCK_HEADER</font>`<font style="color:rgb(55, 55, 55);">) followed by a variable number of cells, which encode keys, values or addresses (see </font>`<font style="color:rgb(55, 55, 55);">cell.h</font>`<font style="color:rgb(55, 55, 55);">).</font>

### Checkpoint:
+ At the block manager level, a checkpoint corresponds with a set of blocks that are stored in the file associated with the given URI.
+ There are three extent lists that are maintained per checkpoint:
    - alloc: The file ranges allocated for a given checkpoint.
    - avail: The file ranges that are unused and available for allocation.
    - discard: The file ranges freed in the current checkpoint.
    - The alloc and discard extent lists are maintained as a skiplist sorted by file offset. The avail extent list also maintains an extra skiplist sorted by the extent size to aid with allocating new blocks.
+ <font style="color:rgb(55, 55, 55);">Each extent uses a file offset and size to track a portion of the file.</font>

## Concept in Memory
```c
struct __wt_btree {
    WT_REF root;                /* Root page reference */
    
    WT_BM *bm;          /* Block manager reference */
};

struct __wt_ref {
    wt_shared WT_PAGE *page; /* Page */

    /*
     * Address: on-page cell if read from backing block, off-page WT_ADDR if instantiated in-memory,
     * or NULL if page created in-memory.
     */
    wt_shared void *addr;
}

/* Block manager handle, references a single checkpoint in a btree. */
struct __wt_bm {
    WT_BLOCK *block;   
}

struct __wt_block {
    WT_FH *fh;            /* Backing file handle */
    wt_off_t size;        /* File size */

    WT_BLOCK_CKPT live;    /* Live checkpoint */
}

struct __wt_connection_impl {
    /* Block objects can be shared (although there can be only one writer). */
    WT_SPINLOCK block_lock; /* Locked: block manager list */
    TAILQ_HEAD(__wt_blockhash, __wt_block) * blockhash;  
    TAILQ_HEAD(__wt_block_qh, __wt_block) blockqh;
    
    WT_BLKCACHE blkcache;     /* Block cache */
    WT_CHUNKCACHE chunkcache; /* Chunk cache */

    WT_CACHE *cache;                        /* Page cache */
    wt_shared volatile uint64_t cache_size; /* Cache size (either statically configured or the current size within a cache pool). */
}

struct __wt_page {
    /* Internal pages */
    wt_shared WT_PAGE_INDEX *volatile __index; /* Collated children */

    /* Row-store leaf page. */
    WT_ROW *row;  /* Key/value pairs */

    /* Page's on-disk representation: NULL for pages created in memory. */
    const WT_PAGE_HEADER *dsk;

    /* If/when the page is modified, we need lots more information. */
    wt_shared WT_PAGE_MODIFY *modify;
}


```

## Summary:
![画板](https://cdn.nlark.com/yuque/0/2024/jpeg/47415822/1732452981281-98f0dd9c-3b68-4c11-8aa4-68f4d50ef8a8.jpeg)

![](https://cdn.nlark.com/yuque/0/2024/png/47415822/1732440459138-cf5bfd4e-72eb-4640-8c0c-d8cab1fd9ead.png)

## Flow:
```c
int __wt_btree_open(WT_SESSION_IMPL *session, const char *op_cfg[]) {}

static int __page_read(WT_SESSION_IMPL *session, WT_REF *ref, uint32_t flags) {}

static int __bm_write(WT_BM *bm, WT_SESSION_IMPL *session, WT_ITEM *buf, 
                      uint8_t *addr, size_t *addr_sizep,
                      bool data_checksum, bool checkpoint_io) {}
```

<font style="color:rgb(55, 55, 55);"></font>

<font style="color:rgb(55, 55, 55);"></font>



