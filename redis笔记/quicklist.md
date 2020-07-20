# quicklist（A generic doubly linked quicklist）
quicklist是一个双向链表，其中每一个节点都是一个压缩列表（ziplist）。

## 数据结构

```c
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor.
 * 'bookmakrs are an optional feature that is used by realloc this struct,
 *      so that they don't consume memory when not used. */
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;    // 总元素数（包含了ziplist）/* total count of all entries in all ziplists */ 
    unsigned long len;      // quicklist的长度        /* number of quicklistNodes */  
    int fill : QL_FILL_BITS;// 每个节点最多包含的数据长度（负数为字节限制，正数为元素长度）              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS;   // 双端有n个节点不压缩 /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;


/* quicklistNode is a 32 byte struct describing a ziplist for a quicklist.
 * We use bit fields keep the quicklistNode at 32 bytes.
 * count: 16 bits, max 65536 (max zl bytes is 65k, so max count actually < 32k).
 * encoding: 2 bits, RAW=1, LZF=2.
 * container: 2 bits, NONE=1, ZIPLIST=2.
 * recompress: 1 bit, bool, true if node is temporarry decompressed for usage.
 * attempted_compress: 1 bit, boolean, used for verifying during testing.
 * extra: 10 bits, free for future use; pads out the remainder of 32 bits */
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;           // ziplist的实际数据
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```


## 新增尾节点
```c
/* Add new entry to tail node of quicklist.
 *
 * Returns 0 if used existing tail.
 * Returns 1 if new tail created. */
int quicklistPushTail(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_tail = quicklist->tail;
    // 尾节点的ziplist，是否有足够的空间插入新值
    // 有：插入；没有：新增一个尾节点
    if (likely(
            _quicklistNodeAllowInsert(quicklist->tail, quicklist->fill, sz))) {
        quicklist->tail->zl =
            ziplistPush(quicklist->tail->zl, value, sz, ZIPLIST_TAIL);
        quicklistNodeUpdateSz(quicklist->tail); // 更新节点的sz为ziplist的实际大小
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_TAIL);

        quicklistNodeUpdateSz(node); // 更新节点的sz为ziplist的实际大小
        _quicklistInsertNodeAfter(quicklist, quicklist->tail, node); //1. 更新链表节点。2.新节点永远不会压缩，旧节点(旧尾节点)判断是否需要压缩。3. 更新链表记录的长度
    }
    quicklist->count++;
    quicklist->tail->count++;
    return (orig_tail != quicklist->tail);
}
```

## 总结
- 本质是一个双端链表，其中每个节点为ziplist
- 双端N个节点的ziplist不会压缩，其余会压缩。（N = quicklist.compress）
- 有一个类似hashmap的问题，如果ziplist的容量过大，则quicklist趋向于一个ziplist；如果ziplist的容量过小，则quicklist趋向于一个普通双端链表。fill的字段用于调节ziplist最大容量
- 直接使用双端链表，产生的碎片会比较多
- 直接使用ziplist，每次增删元素可能会导致连锁更新，开销较大