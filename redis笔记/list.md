# 双向链表
adlist、ziplist、quicklist

## 一、ADLIST（A generic doubly linked list）数据结构
```c
/* Node, List, and Iterator are the only data structures used currently. */

typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;          // 迭代器方向， 0:从头开始，1:从尾开始
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);            // 节点复制
    void (*free)(void *ptr);            // 节点释放
    int (*match)(void *ptr, void *key); // 节点对比
    unsigned long len;                  //记录长度
} list;
```