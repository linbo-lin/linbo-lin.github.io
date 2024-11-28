+++
title = 'WiredTiger - Cache相关概念梳理'
date = 2024-11-29T00:17:04+08:00
tags = []
draft = false
+++

## Concept on Disk
### Block Manager:
[https://source.wiredtiger.com/11.3.1/arch-block.html](https://source.wiredtiger.com/11.3.1/arch-block.html)

- A block is a chunk of data that is stored on the disk and operated on as a single unit.
- Each WiredTiger data file (any file in the home directory with the .wt suffix) is made up of these blocks.
- Each block consists of a page header, a block header and contains a single page of the btree from which it was generated.
- WiredTiger is a no-overwrite storage engine, and when blocks are re-written, they are written to new locations in the file.

{{< figure src="../images/block.png" title="" align="center">}}

### Descriptor blocks:
- A file is divided up into blocks. The first block in a file is special as it contains metadata about the file and is referred to as the "descriptor block". It contains the WiredTiger major and minor version, a checksum of the block contents as well as a "magic" number to check against.

### Data File Format:
[https://source.wiredtiger.com/11.3.1/arch-data-file.html](https://source.wiredtiger.com/11.3.1/arch-data-file.html)

Btree <----> Table 1 : 1
- In memory, a database table is represented by a B-Tree data structure, which is made up of nodes that are page structures.
- The root and internal pages will only store keys and references to other pages, while leaf pages store keys, values and sometimes references to overflow pages.

Block <----> Page  1 : 1
- When these pages are written onto the disk, they are written out as units of data called blocks.
- On the disk, a WiredTiger data file is just a collection of blocks which logically represent the pages of the B-Tree.

Data File --- Block --- Page on Disk
- Each WiredTiger data file (any file in the home directory with the .wt suffix) is made up of these blocks.
- On the disk, a WiredTiger data file is just a collection of blocks which logically represent the pages of the B-Tree.
- The layout of a .wt file consists of a file description WT_BLOCK_DESC which always occupies the first block, followed by a set of on-disk pages. 

### Page on Disk
- Pages consist of a header (WT_PAGE_HEADER and WT_BLOCK_HEADER) followed by a variable number of cells, which encode keys, values or addresses (see cell.h).

### Checkpoint:
- At the block manager level, a checkpoint corresponds with a set of blocks that are stored in the file associated with the given URI. 
- There are three extent lists that are maintained per checkpoint:
  - alloc: The file ranges allocated for a given checkpoint. 
  - avail: The file ranges that are unused and available for allocation. 
  - discard: The file ranges freed in the current checkpoint. 
  - The alloc and discard extent lists are maintained as a skiplist sorted by file offset. The avail extent list also maintains an extra skiplist sorted by the extent size to aid with allocating new blocks.
- Each extent uses a file offset and size to track a portion of the file.

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

## Summary
{{< figure src="../images/cache.jpg" title="" align="center">}}

{{< figure src="../images/page.png" title="" align="center">}}

## Flow

```c
int __wt_btree_open(WT_SESSION_IMPL *session, const char *op_cfg[]) {}

static int __page_read(WT_SESSION_IMPL *session, WT_REF *ref, uint32_t flags) {}

static int __bm_write(WT_BM *bm, WT_SESSION_IMPL *session, WT_ITEM *buf, 
                      uint8_t *addr, size_t *addr_sizep,
                      bool data_checksum, bool checkpoint_io) {}
```
