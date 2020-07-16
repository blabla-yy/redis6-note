# SDS（Simple Dynamic String）

## 一、特点

1. 支持O(1)获取长度（__len字段__）
2. __空间预分配__，额外分配的空间，字符串操作时会检查、使用、扩容此空间。
3. 惰性释放空间
4. __二进制安全__，避免出现因二进制中存在'\0'，导致string.h错误截断字符串
5. 兼容部分C字符串函数。（同样以\0结尾）
6. __压缩空间__：尽量不使用内存对齐；使用五个不同结构体存储不同长度的内容（len和alloc字段长度不同）

## 二、sds的五种结构体（3.2+）
sdshdr5，sdshdr8，sdshdr16，sdshdr32，sdshdr64

```c
// NOTE: 3.2以前旧版本中仅有一个类型：
// struct sdshdr {
//     unsigned int len;//表示sds当前的长度
//     unsigned int free;//已为sds分配的长度-sds当前的长度
//     char buf[];//sds实际存放的位置
// };

// 当前版本
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};

// __packed：GCC编译器尽量不使用内存对齐
// 16、32、64与8类似，只是len和alloc是uintN_t
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
1. 五个类型根据sds的实际长度判断的，不同sdshdr内存空间不同，可以节省内存
2. flags字段一共8位，低3位是存储当前类型（000: sdshdr5、001:sdshdr8以此类推，所以宏SDS_TYPE_5=0，以此类推），除sdshdr5，其余结构体的flags高五位不使用
3. sdshdr5 flags的高五位用来存储长度，所以最大长度1 << 5 = 32位
4. len：长度（字节），不包含最后的'\0'
5. alloc：预分配的空间（字节）
6. buf：存储的数据


判断当前sds的结构体类型

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
// 根据sds，创建结构体指针sh
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
// 根据sds，获取结构体的匿名变量
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)

/**
根据sds获取结构体的实际长度
1.结构体中sds的指针当前移动是flags
2.SDS_TYPE_MASK： 0111， 所以flags & SDS_TYPE_MASK： = 取flags的低三位
**/
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

> sdshdr16结构  
> len(2byte) | alloc(2byte) | flags(1byte) | buf(bytes)  
> 所以数据(sds)的指针 - 1是flags

## 三：sdshdr初始化与内存分配

1. 根据长度判断应该创建哪个结构体
```c
// 根据长度判断类型
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
// 64位系统long int占8字节与longlong int相同，
// 32位long int占4字节，longlong int 8字节
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}
```

2. 根据结构体和数据长度，申请内存（zmalloc）,长度为hdrlen + initlen + 1（结构体、数据、\0）
3. 赋值结构体成员

核心代码：

```c
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 * If SDS_NOINIT is used, the buffer is left uninitialized;
 *
 * The string is always null-termined (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3);
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
sds sdsnewlen(const void *init, size_t initlen) { // init：需要存储的数据，initlen：数据长度
    void *sh;
    sds s;
    char type = sdsReqType(initlen); // 根据长度判断类型
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type); // 计算结构体长度
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1); // 结构体 + 数据 + \0
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1); // 长度为0时
    s = (char*)sh+hdrlen; // s指向buf
    fp = ((unsigned char*)s)-1; // s - 1 是flags字段
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS); // 高位存储长度，低位存储类型
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

# 四、sds扩展（sdscat）

```c
/**
 * s：当前sds
 * t：需要拼接的内容（binary-safe）
 * len：t的长度
 */ 
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len); // 判断是否需要扩展空间、更换结构体类型
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}

/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s); // 获取剩余空间
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    // 计算新的空间大小
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC) // 小于1MB时
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) { // 类型相同，直接分配
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {  // 类型不同，即header不同，需要重新分配
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}
```