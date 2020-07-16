# 压缩列表（ziplist）

## 数据结构
```text
 [0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
       |             |          |       |       |     |
    zlbytes        zltail    entries   "2"     "5"   end
 ```
 - zlbytes: 列表长度（4字节）
 - zltail: 尾元素的偏移
 - entries: 元素个数
 - 2 / 5: 存储的元素
 - end: 结束位

### entries的数据结构：

```text
<prevlen> <encoding> <entry-data>
或
<prevlen> <encoding>
```
- prevlen: 前一个元素的长度
- encoding: 编码
- 当encoding可以表示本身，如较小的整型，则会去掉entry-data部分



 ```c
 /* We use this function to receive information about a ziplist entry.
 * Note that this is not how the data is actually encoded, is just what we
 * get filled by a function in order to operate more easily. */
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
 ```


## 连锁更新
由于prevlen保存了上一节点的长度，如果上一节点出现长度变化，prevlen当前空间不足以存储，则会发生扩容。发生扩容会，也可能会影响下一节点，进而连锁更新。最差情况O(n)