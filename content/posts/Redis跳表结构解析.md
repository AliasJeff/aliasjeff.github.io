+++
title = "Redis 跳跃表结构解析"
date = "2026-09-07"

[taxonomies]
tags = ["Redis", "数据结构"]

[extra]
comment = true
+++

跳跃表（skiplist）是一种随机化的有序数据结构，由 William Pugh 在论文《Skip lists: a probabilistic alternative to balanced trees》中提出。它通过在有序链表之上增加多层索引，让查找、插入和删除操作的平均复杂度达到 `O(log N)`，实现却比平衡树更直观。

Redis 使用跳跃表作为有序集合（sorted set）的底层实现之一。当有序集合中的元素较多，或者成员字符串较长时，Redis 会使用跳跃表和字典共同保存数据。

## 从有序链表到跳跃表

有序链表中的节点按照递增或递减顺序排列，每个节点通过 `next` 指针指向下一个节点。

假设链表中的元素为：

```text
1 -> 8 -> 11 -> 12 -> 26 -> 31 -> 61
```

如果查找 `31`，只能从头开始逐个比较，查找路径是 `1 -> 8 -> 11 -> 12 -> 26 -> 31`，时间复杂度为 `O(N)`。

如果将部分节点抽出来组成更高层的有序链表，查找时先从最高层开始向后移动；当下一个节点大于目标值，或者已经到达当前层末尾时，再下降一层继续查找，就可以跳过大量节点。

例如查找 `31` 时：

1. 从最高层的 `1` 开始，移动到 `11`；
2. 最高层的下一个节点为空，下降到下一层；
3. 在下一层从 `11` 移动到 `26`；
4. `26` 的下一个节点是 `61`，大于 `31`，继续下降；
5. 在第 0 层从 `26` 移动到 `31`，找到目标节点。

这就是跳跃表的基本思想：底层保存所有元素，上层保存部分元素作为索引，查找过程中通过“向右移动、向下下降”来减少比较次数。

## 跳跃表的基本性质

Redis 的跳跃表具有以下特点：

- 最底层 `Level 0` 包含所有节点；
- 每一层都是按 `score` 递增排列的有序链表；
- 上层出现的节点一定会在下层出现；
- 跳跃表有一个固定最多 32 层的头节点；
- 除头节点外，最高节点的层数决定跳跃表的当前高度 `level`；
- `tail` 指针指向最底层的最后一个节点；
- 每层最后一个节点的 `forward` 都指向 `NULL`。

`level` 表示当前实际使用的最高层数，头节点虽然有 32 层，但头节点本身不计入跳跃表高度。

## 跳跃表结构

Redis 中跳跃表的结构体定义如下：

```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

各字段的含义如下：

- `header`：指向头节点。头节点最多拥有 32 个层级，不保存实际的 `score` 和成员对象；
- `tail`：指向最底层的最后一个节点；
- `length`：第 0 层中实际节点的数量，不包括头节点；
- `level`：除头节点外，当前最高节点的层数。

头节点和普通节点使用相同的节点结构，但头节点的 `score`、成员对象和后退指针不会用于保存实际数据。

## 跳跃表节点

节点定义如下：

```c
typedef struct zskiplistNode {
    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

### `score`

`score` 是一个 `double` 类型的分值，跳跃表中的节点首先按照 `score` 从小到大排序。

### `obj`

`obj` 是节点的成员对象，指向一个 Redis 字符串对象。字符串对象底层通常保存着一个 SDS。

当多个节点的 `score` 相同时，Redis 会继续比较成员对象的字典序，以确定节点顺序。

### `backward`

`backward` 是后退指针，只存在于节点的底层关系中，用于从表尾向表头方向遍历。头节点和第一个实际节点的 `backward` 都指向 `NULL`。

### `level`

`level` 是一个动态柔性数组。每个节点根据随机算法获得不同的高度，数组中的每个元素保存一个 `forward` 指针和一个 `span`：

- `forward`：指向当前层的下一个节点；
- `span`：记录从当前节点到 `forward` 节点之间跨越的节点数量。

`span` 不仅能支持跳跃式查找，还能帮助 Redis 根据排名查找元素。

## 随机生成节点高度

每插入一个新节点，Redis 都会随机为它分配一个高度。节点高度一旦确定，后续不会再改变。

```c
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

int zslRandomLevel(void) {
    int level = 1;

    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;

    return (level < ZSKIPLIST_MAXLEVEL)
        ? level
        : ZSKIPLIST_MAXLEVEL;
}
```

这里 `level` 从 1 开始。每次随机结果满足条件，就晋升到更高一层；否则停止。Redis 将最大层数限制为 32，避免节点层级无限增长。

由于 `ZSKIPLIST_P` 为 `0.25`，严格按照这段代码，节点继续向上一层晋升的概率约为四分之一。因此节点高度越高，出现的概率越低，跳跃表整体会保持比较稀疏的高层索引。

## 创建跳跃表

创建跳跃表主要分为三步：

1. 分配并初始化 `zskiplist`；
2. 设置 `level = 1`、`length = 0`，并令 `tail = NULL`；
3. 创建一个拥有 32 层的头节点，将每一层的 `forward` 和 `span` 初始化为 0 或 `NULL`。

```c
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn) + level * sizeof(struct zskiplistLevel));

    zn->score = score;
    zn->obj = obj;
    return zn;
}

zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl = zmalloc(sizeof(*zsl));

    zsl->level = 1;
    zsl->length = 0;

    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL, 0, NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }

    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

## 插入节点

插入一个节点时，Redis 会完成以下工作：

1. 从最高层开始查找插入位置，并记录每一层最后访问的节点；
2. 记录查找过程中跨越的节点数量；
3. 随机生成新节点的高度，必要时增加跳跃表的当前高度；
4. 修改新节点和前驱节点的 `forward`；
5. 更新相关节点的 `span`；
6. 设置新节点的 `backward`，必要时更新 `tail`；
7. 增加 `length`。

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    x = zsl->header;
    for (i = zsl->level - 1; i >= 0; i--) {
        rank[i] = (i == zsl->level - 1) ? 0 : rank[i + 1];

        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 compareStringObjects(x->level[i].forward->obj, obj) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }

        update[i] = x;
    }

    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }

    x = zslCreateNode(level, score, obj);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        x->level[i].span =
            update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    for (i = level; i < zsl->level; i++)
        update[i]->level[i].span++;

    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;

    zsl->length++;
    return x;
}
```

### 为什么要维护 `update` 和 `rank`

`update[i]` 保存第 `i` 层中，新节点应该插入位置的前驱节点。插入时，Redis 可以直接修改这些节点的指针，不需要再次遍历。

`rank[i]` 保存从头节点到当前位置已经跨过的节点数。插入节点后，Redis 根据不同层级的距离重新计算 `span`，这样才能继续支持按排名查找。

## 删除节点

真正删除节点时，需要修复所有受影响层级的 `forward` 和 `span`，然后更新后退指针、尾指针、跳跃表高度和节点数量。

```c
void zslDeleteNode(zskiplist *zsl,
                   zskiplistNode *x,
                   zskiplistNode **update) {
    int i;

    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }

    if (x->level[0].forward)
        x->level[0].forward->backward = x->backward;
    else
        zsl->tail = x->backward;

    while (zsl->level > 1 &&
           zsl->header->level[zsl->level - 1].forward == NULL)
        zsl->level--;

    zsl->length--;
}
```

如果 `update[i]` 的 `forward` 正好指向待删除节点 `x`，就把它改成 `x` 的后继节点，并将原有跨度合并；否则说明这一层没有直接经过 `x`，只需要把跨度减一。

删除节点后，如果最高层已经没有实际节点，Redis 会降低跳跃表的当前高度，但不会缩小头节点固定的 32 层数组。

## 删除指定成员

`zslDelete` 先按照 `score` 和成员对象的字典序查找节点，同时记录各层的前驱节点。找到同时满足 `score` 和成员对象的节点后，调用 `zslDeleteNode` 完成删除。

```c
int zslDelete(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level - 1; i >= 0; i--) {
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 compareStringObjects(x->level[i].forward->obj, obj) < 0))) {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    x = x->level[0].forward;
    if (x && score == x->score && equalStringObjects(x->obj, obj)) {
        zslDeleteNode(zsl, x, update);
        zslFreeNode(x);
        return 1;
    }

    return 0;
}
```

同一个 `score` 可以对应多个成员，所以只比较分值是不够的，必须同时比较成员对象。

## 常用 API 与复杂度

| 函数 | 作用 | 平均复杂度 |
| --- | --- | --- |
| `zslCreate` | 创建跳跃表 | `O(1)` |
| `zslFree` | 释放跳跃表及所有节点 | `O(N)` |
| `zslInsert` | 插入节点 | `O(log N)` |
| `zslDelete` | 删除指定节点 | `O(log N)` |
| `zslFirstInRange` | 返回范围内第一个节点 | `O(log N)` |
| `zslLastInRange` | 返回范围内最后一个节点 | `O(log N)` |
| `zslIsInRange` | 判断分值范围是否可能命中 | `O(1)` |
| `zslGetRank` | 获取节点排名 | `O(log N)` |
| `zslGetElementByRank` | 按排名获取节点 | `O(log N)` |
| `zslDeleteRangeByScore` | 按分值范围删除节点 | `O(log N + N)` |
| `zslDeleteRangeByRank` | 按排名范围删除节点 | `O(log N + N)` |

实际实现中，复杂度还会受到成员字符串比较成本的影响；当需要比较很长的成员字符串时，单次比较本身也可能消耗额外时间。

## 总结

跳跃表的核心并不在于某个复杂的平衡算法，而在于“多层有序链表”这一简单结构：

- 第 0 层保存全部元素，保证顺序遍历；
- 更高层保存部分节点，作为稀疏索引；
- 随机高度使高层节点逐渐变少；
- `forward` 支持向后查找，`backward` 支持反向遍历；
- `span` 支持按排名定位；
- 插入和删除只需要更新受影响路径上的节点。

Redis 选择跳跃表，除了平均复杂度接近平衡树之外，也因为它的实现更容易和范围查询、排名查询结合。对于有序集合这种既需要按分值排序，又需要频繁插入、删除和按排名访问的场景，跳跃表是一个很实用的工程选择。
