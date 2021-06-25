---
title: Redis-Hash-数据结构
date: 2020-11-13 20:30:01
tags: [Redis]
description: Redis Hash数据结构源码分析
---

# Redis-HashTable-数据结构

## 目录

- [Redis-HashTable-接口](https://lrtz-v.github.io/2020/11/13/Redis-HashTable-Api/)

## 源码

- [dict.h](https://github.com/redis/redis/blob/unstable/src/dict.h)
- [dict.c](https://github.com/redis/redis/blob/unstable/src/dict.c)

## dictType

- 定义一组 key-value 的操作函数模版

```c
typedef struct dictType {
    // 计算hash的函数
    uint64_t (*hashFunction)(const void *key);

    // key 复制函数，一般使用NULL
    void *(*keyDup)(void *privdata, const void *key);

    // value 复制函数，一般使用NULL
    void *(*valDup)(void *privdata, const void *obj);

    // key 比较函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // key 释放函数
    void (*keyDestructor)(void *privdata, void *key);

    // value 释放函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

## dict - 字典结构

```c
typedef struct dict {
    // key-value 操作函数组
    dictType *type;

    // 私有数据，用在特定 hooks 函数中
    void *privdata;

    // hashTable, ht[1] 扩容中的hashTable, ht[0] 旧的hashTable
    dictht ht[2];

    // rehash 标志，默认为-1（未在扩容中）
    long rehashidx;

    // 迭代器数量
    unsigned long iterators;
} dict;
```

## dictht - hashTable 结构

```c
typedef struct dictht {
    // hash 数组
    dictEntry **table;

    // hash 数组大小
    unsigned long size;

    // 哈希表大小掩码，等于size-1
    unsigned long sizemask;

    // hash 数组已使用大小
    unsigned long used;
} dictht;
```

## dictEntry - hash 节点

```c
typedef struct dictEntry {
    // 元素 key
    void *key;

    // 元素 value
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;

    // 下一个节点的地址
    struct dictEntry *next;
} dictEntry;
```

## 迭代器

```c
typedef struct dictIterator {
    // dict
    dict *d;

    //
    long index;

    // hashTable 标志(0/1)；安全标志
    int table, safe;

    // 当前entrey、链表的下一个节点
    dictEntry *entry, *nextEntry;

    // dict 指纹
    long long fingerprint;
} dictIterator;
```
