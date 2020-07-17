# 整型集合(intset)

## 数据结构

```c
typedef struct intset {
    uint32_t encoding;  // 编码
    uint32_t length;    // 长度
    int8_t contents[];  // 内容
} intset;

// 共三种encoding
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))

// 根据值进行encoding判断
static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX)
        return INTSET_ENC_INT32;
    else
        return INTSET_ENC_INT16;
}
```

## 查询节点
```c
/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted. */
/**
 *  is: set
 *  value: 查询的值
 *  pos: value值的位置。
 *  
 *  return 0：value不存在，pos时可以插入的位置。1：value存在，pos此值的位置
 */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    // 集合长度位0，则肯定不存在value值，位置位0
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        // 如果查询的值，大于最后一个位置的值 或者小于第一个位置的值，则可以判断此值不存在，并赋值pos
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
    // 二分查找
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        // 最后一次循环，mid值将会等于min。如果value > mid，min后移导致退出，则位置就是min。同理value < mid，max前移导致退出，位置仍是min
        if (pos) *pos = min; 
        return 0;
    }
}
```

## 新增节点

```c
/* Insert an integer in the intset */
/**
 * is: 集合
 * value: 需要新增的值
 * success: 操作标识位（set集合，新增重复的值会失败） 
 */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;  // success初始化默认值

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) { // 新值的编码导致整体编码大小变化，则需要更新set的空间长度、所有值的编码
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {  // 判断是否重复值并且，获取新值的位置赋值pos
            if (success) *success = 0;      // 重复值时：success = false，返回原set
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);                 // 增加空间
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); // pos位置到尾端的值全部向后移动
    }

    // 插入value到pos，更新length
    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

// intset的编码变化，将更新所有值，并新增节点
/* Upgrades the intset to a larger encoding and inserts the given integer. */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);    // 当前编码
    uint8_t newenc = _intsetValueEncoding(value);   // 新编码
    int length = intrev32ifbe(is->length);          // 长度
    int prepend = value < 0 ? 1 : 0;                // 插入头节点 / push节点

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);   // 整体长度重分配（zrealloc）

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    // 由后至前，将原值重新编码后置入set
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

## 总结
- 自增的整型序列
- Set集合，不会出现重复值
- 为节省空间，选择尽量以较少空间的类型进行编码，如果此编码类型空间不足，则会扩容。