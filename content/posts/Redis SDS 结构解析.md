+++
title = "Redis SDS 结构解析"
date = "2026-09-19"

[taxonomies]
tags = ["Redis", "数据结构", "C语言"]

[extra]
comment = true
+++

Redis 没有直接使用 C 语言的字符数组来表示字符串，而是自己实现了一套简单动态字符串（Simple Dynamic String，SDS）。SDS 既保留了 C 字符串使用方便的特点，又解决了传统 C 字符串在长度获取、二进制数据处理和频繁修改时的几个问题。

本文从 SDS 的结构开始，结合源码说明 Redis 是如何完成字符串存储、扩容和缩短操作的。

## 什么是 SDS

传统 C 字符串本质上是一个以空字符 `\0` 结尾的字符数组。它有一个天然限制：如果字符串内容本身包含 `\0`，C 字符串就会把它当成结束位置，后面的内容无法继续读取。这也是为什么 C 字符串不是二进制安全的。

Redis 中，C 字符串主要用于字符串字面量，例如日志内容和错误信息。只要字符串需要被保存或修改，Redis 通常就会使用 SDS。

例如客户端执行：

```text
redis> SET msg "hello world"
OK
```

Redis 会创建一个键值对：

- 键是一个字符串对象，底层 SDS 保存着 `"msg"`；
- 值也是一个字符串对象，底层 SDS 保存着 `"hello world"`。

## 为什么使用 SDS

SDS 主要解决了传统 C 字符串的几个问题。

### 常数时间获取长度

SDS 在头部保存字符串长度，不需要像 `strlen` 一样从头扫描到 `\0`，获取长度的复杂度是 `O(1)`。

### 二进制安全

SDS 通过长度字段判断有效内容，读写时不依赖 `\0`。因此即使字符串中间包含空字符，也不会被截断。

### 兼容 C 字符串

SDS 对上层暴露的不是头部结构体指针，而是直接指向字符数组 `buf` 的指针。`buf` 末尾仍然会保留一个 `\0`，因此保存文本时可以直接使用一部分 C 标准库字符串函数。

### 减少内存重分配

SDS 会在扩容时预留一部分空闲空间，在缩短字符串时也不会立即释放多余空间。这样可以减少连续追加或修改带来的内存分配次数。

### 使用柔性数组连续存储

字符串内容存放在结构体末尾的柔性数组中。头部和字符串内容处在同一块连续内存里，不需要再通过额外指针定位字符串，访问和管理都比较直接。

## SDS 的数据结构

现代 Redis 的 SDS 头部包含以下信息：

- `len`：`buf` 中已经使用的字节数；
- `alloc`：为 `buf` 分配的总容量，不包括头部和结尾的空字符；
- `flags`：标记头部类型，低 3 位用于保存类型；
- `buf[]`：实际保存字符串内容的柔性数组。

Redis 根据字符串长度选择不同大小的头部类型：

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

各类型的头部大小如下：

| 类型 | 长度范围 | `len` / `alloc` 字段 | 头部大小 |
| --- | --- | --- | --- |
| `sdshdr5` | `[0, 1 << 5)` | 没有独立字段，长度放在 `flags` 高 5 位 | 1 字节 |
| `sdshdr8` | `[1 << 5, 1 << 8)` | 各 1 字节 | 3 字节 |
| `sdshdr16` | `[1 << 8, 1 << 16)` | 各 2 字节 | 5 字节 |
| `sdshdr32` | `[1 << 16, 1 << 32)` | 各 4 字节 | 9 字节 |
| `sdshdr64` | `[1 << 32, 1 << 64)` | 各 8 字节 | 17 字节 |

SDS 本质上就是一个 `char *`，它指向 `buf` 的起始位置。头部位于这个指针之前，Redis 可以通过指针向前偏移找到头部，再根据 `flags` 判断当前使用的是哪一种头部。

`sdshdr5` 是一个特殊情况：它没有单独的 `len` 和 `alloc` 字段，而是使用 `flags` 的高 5 位保存字符串长度。由于它无法记录额外的空闲空间，追加内容时通常会切换到 `sdshdr8` 或更大的类型。

## 一个简化版 SDS

早期 Redis 版本中的 SDS 结构可以简化为下面这样：

```c
typedef char *sds;

struct sdshdr {
    // buf 中已经使用的空间长度
    int len;
    // buf 中剩余可用空间长度
    int free;
    // 数据空间
    char buf[];
};
```

实际 Redis 源码使用了 `sdshdr5`、`sdshdr8`、`sdshdr16`、`sdshdr32` 和 `sdshdr64` 多种头部，以便在字符串较短时节省内存。上面的结构更适合理解 SDS 的基本思想。

### 获取长度和剩余空间

```c
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void *)(s - sizeof(struct sdshdr));
    return sh->len;
}

static inline size_t sdsavail(const sds s) {
    struct sdshdr *sh = (void *)(s - sizeof(struct sdshdr));
    return sh->free;
}
```

这里利用了柔性数组和结构体连续分配的特性：`s` 指向 `buf`，向前移动一个头部大小，就可以得到头部地址。

实际版本中，Redis 会先读取 `s[-1]` 得到 `flags`，再根据头部类型计算对应的头部大小。无论哪一种实现，核心思路都是一样的。

## 创建 SDS

`sdsnewlen` 根据初始内容和长度分配一块连续内存，完成头部初始化后返回 `buf` 的地址，而不是返回头部地址。

```c
sds sdsnewlen(const void *init, size_t initlen)
{
    struct sdshdr *sh;

    if (init) {
        // zmalloc 不初始化分配出来的内存
        sh = zmalloc(sizeof(struct sdshdr) + initlen + 1);
    } else {
        // zcalloc 会把分配的内存初始化为 0
        sh = zcalloc(sizeof(struct sdshdr) + initlen + 1);
    }

    if (sh == NULL)
        return NULL;

    sh->len = initlen;
    sh->free = 0;

    if (initlen && init)
        memcpy(sh->buf, init, initlen);

    // 保留 C 字符串结尾的空字符
    sh->buf[initlen] = '\0';

    // 返回 buf，而不是整个头部
    return (char *)sh->buf;
}
```

这里最后多分配的 1 个字节用于保存 `\0`。它不会参与 `sdslen` 返回的长度，因此不会影响 SDS 的二进制安全特性。

## 空间预分配

对 SDS 进行追加或拷贝时，Redis 通常会先调用 `sdsMakeRoomFor` 检查剩余空间。

如果当前空闲空间足以容纳新增内容，函数直接返回，不发生内存分配。否则，Redis 会按照下面的策略扩容：

- 新长度小于 `1 MB` 时，分配约为新长度两倍的空间；
- 新长度大于等于 `1 MB` 时，在新长度之外额外预留 `1 MB`。

这是一种空间换时间的做法。短字符串追加时会有较明显的预留，长字符串则把额外空间控制在固定大小，避免一次性浪费过多内存。

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    // 空间足够，直接返回
    if (avail >= addlen)
        return s;

    len = sdslen(s);
    sh = (char *)s - sdsHdrSize(oldtype);
    newlen = len + addlen;

    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    // sdshdr5 无法记录空闲空间，追加时不能继续使用它
    if (type == SDS_TYPE_5)
        type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);

    if (oldtype == type) {
        // 头部大小不变，可以直接 realloc
        newsh = s_realloc(sh, hdrlen + newlen + 1);
        if (newsh == NULL)
            return NULL;

        s = (char *)newsh + hdrlen;
    } else {
        // 头部类型变化，头部大小也变化，需要重新分配并搬迁数据
        newsh = s_malloc(hdrlen + newlen + 1);
        if (newsh == NULL)
            return NULL;

        memcpy((char *)newsh + hdrlen, s, len + 1);
        s_free(sh);

        s = (char *)newsh + hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }

    sdssetalloc(s, newlen);
    return s;
}
```

这里有一个容易忽略的细节：字符串变长后，原来的头部类型可能无法存储新长度。例如 `sdshdr8` 无法继续表示超过 255 字节的容量。这时不仅要扩容，还要更换头部类型。

如果头部类型没有变化，可以使用 `realloc`；如果头部大小变化，就必须申请新内存、复制字符串，再释放旧内存。

## 惰性空间释放

SDS 缩短时，Redis 通常不会立即释放多出来的空间，而是把它记录为空闲空间，留给后续追加操作使用。

例如 `sdsclear` 会把字符串清空，但不会释放原来的缓冲区：

```c
void sdsclear(sds s)
{
    struct sdshdr *sh = (void *)(s - sizeof(struct sdshdr));

    sh->free += sh->len;
    sh->len = 0;
    sh->buf[0] = '\0';
}
```

这个操作的复杂度是 `O(1)`。下一次追加内容时，只要不超过原有容量，就可以直接写入，不需要重新分配内存。

## 总结

SDS 的结构并不复杂，但它把 C 字符串几个常见的痛点处理得很实用：

- 用 `len` 或等价字段在 `O(1)` 时间内获取长度；
- 通过长度而不是 `\0` 判断有效内容，保证二进制安全；
- 保留末尾的 `\0`，兼容一部分 C 字符串函数；
- 通过空间预分配减少频繁扩容；
- 通过惰性空间释放减少缩短字符串时的内存操作；
- 通过多种头部类型，在功能和内存占用之间做平衡。

SDS 对外表现为一个指向 `buf` 的字符指针，所以使用起来像 C 字符串；但 Redis 真正读取内容时依赖的是头部记录的长度，而不是结尾的 `\0`。这正是 SDS 兼容性、二进制安全和性能之间的平衡点。
