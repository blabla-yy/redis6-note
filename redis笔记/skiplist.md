# 跳跃表

## 数据结构
```c
// 最多32层
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;                            // 成员
    double score;                       // 分数
    struct zskiplistNode *backward;     // 后一个节点
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前节点
        unsigned long span;             // 跨度
    } level[];                          // 层级
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;// 头尾指针
    unsigned long length;               // 节点长度
    int level;                          // 最大层数
} zskiplist;

typedef struct zset {
    dict *dict;                         // O(1)查找元素
    zskiplist *zsl;                     // 根据score查询
} zset;
```


## 新增节点
```c
/* Insert a new node in the skiplist. Assumes the element does not already
 * exist (up to the caller to enforce that). The skiplist takes ownership
 * of the passed SDS string 'ele'. */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    // 从最高层开始遍历
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 当前节点存在forward，且（
        //     forward的分数小于查询的分数 或 （forward分数等于查询的分数 且forward存储对象小于当前）
        //     ）

        // 查找分数位置，如果分数相同，小的在前面。
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();
    if (level > zsl->level) { // 大于目前的最大层级时，更新header的层级，新增的层级插入空值
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }

    // 构造节点，及其level
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 如果新增的节点的level过小，需要补齐level
    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 关联节点
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```


## 不选用红黑树
1. 简单，非特殊情况下，时间复杂度相同
2. 可以 __顺序遍历__ （前后指针）