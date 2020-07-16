# dict

## 特点
1. 链地址法解决冲突
2. 字典dict拥有两个哈希表dictht，一个用来存储数据，另一个rehash
3. 渐进式rehash，用rehashidx进行标记正在进行rehash操作的节点

## 数据结构
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

// 哈希表
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;      // Entry数组
    unsigned long size;     // 数组大小
    unsigned long sizemask; // 数组长度掩码，用于计算索引值，size-1
    unsigned long used;     // 现有节点
} dictht;

// 字典
typedef struct dict {
    dictType *type; // 处理函数
    void *privdata; // 处理函数的私有数据
    dictht ht[2];   // 哈希表，0是主要是用的，1是rehash用的
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */ // rehash的当前节点索引，其余操作可以通过这个索引信息协助rehash。
    unsigned long iterators; /* number of iterators currently running */
} dict;

```

## 渐进式rehash（incremental rehashing）
1. 字典维护了一个rehashidx字段
2. 开始rehash时，rehashidx为0，并且对索引为0的数据进行
3. 外层循环：根据rehashidx进行rehash，并且rehashidx++。结束条件：执行的次数n，或完成rehash。
4. 内层循环：根据rehashidx进行rehash时会跳过空节点，同时有一个 __empty_visits__ 字段避免此循环过久。
5. 基础的增、删、查询操作都会执行以此_dictRehashStep，即执行一次rehash操作协助rehash。

```c
/** 
 * d：字典
 * n：执行次数
 * return：0 已经完成rehash；1 进行中
 */
int dictRehash(dict *d, int n) {
    // empty_visits避免过多空节点，导致内部跳过空节点的循环时间太久
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;  // 已结束则退出

    // 循环进行rehash，执行n次或rehash完成
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) { // 跳过空数据，rehashidx++
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];  // 当前需要移动的节点
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++; // 下一个需要移动的节点
    }

    // 如果结束则释放
    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}



// 扩展时会判断是否正在rehash，不会冲突
/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```