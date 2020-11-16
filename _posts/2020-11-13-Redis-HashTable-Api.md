---
title: Redis-Hash-接口
date: 2020-11-13 20:30:01
tags: [Redis]
---

# Redis-HashTable-接口

## dict 创建 - dictCreate

```c

// 创建新的dict
dict *dictCreate(dictType *type, void *privDataPtr)
{
    // 分配初始内存空间
    dict *d = zmalloc(sizeof(*d));

    // 初始化
    _dictInit(d,type,privDataPtr);

    // 返回dict指针
    return d;
}

// dict 初始化
int _dictInit(dict *d, dictType *type, void *privDataPtr)
{
    // hashtable 初始化
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}

static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

## key 插入 1：dictAddRaw

- 查询 key 的 bucket 并插入 dictEntry
- 当 key 已存在时，抛出异常

```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    // rehash检查及协助
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 计算key的插入索引，如果key已存在，则返回-1
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // 判断操作的hashtable，rehash中的插入操作目标是ht[1]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];

    // 新节点内存分配
    entry = zmalloc(sizeof(*entry));

    // 拉链法解决冲突，首节点插入
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    // 设置value
    dictSetKey(d, entry, key);
    return entry;
}

// 节点 bucket 定位
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    // 扩容检查
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    for (table = 0; table <= 1; table++) {
        // 计算index
        idx = hash & d->ht[table].sizemask;
        he = d->ht[table].table[idx];

        // key冲突时，需要比较链表中的key，判断key是否已存在
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }

        // 不在rehash过程中时，只在ht[0]中定位 bucket 即可
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}


// 扩容检查
static int _dictExpandIfNeeded(dict *d)
{
    // rehash检查
    if (dictIsRehashing(d)) return DICT_OK;

    // 未初始化检查，完成初始化
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    // 当已使用节点大于hashtable大小，并且允许扩容或者使用节点大于hashtable大小的5倍时，允许扩容
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

## key 插入 2：dictAddOrFind

```c
dictEntry *dictAddOrFind(dict *d, void *key) {
    dictEntry *entry, *existing;
    // key插入
    entry = dictAddRaw(d,key,&existing);

    // key不存在，返回新增的entry；否则返回已存在entry
    return entry ? entry : existing;
}
```

## key-value 插入：dictAdd

```c
int dictAdd(dict *d, void *key, void *val)
{
    // key 插入
    dictEntry *entry = dictAddRaw(d,key,NULL);

    // 判断key已存在
    if (!entry) return DICT_ERR;

    // set value
    dictSetVal(d, entry, val);
    return DICT_OK;
}
```

## 插入或替换：dictReplace

```c
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, *existing, auxentry;

    // 当key不存在时插入新节点，否则返回已存在节点
    entry = dictAddRaw(d,key,&existing);
    // key 不存在，set value
    if (entry) {
        dictSetVal(d, entry, val);
        return 1;
    }

    // key 已存在，替换value，释放旧值
    auxentry = *existing;
    dictSetVal(d, existing, val);
    dictFreeVal(d, &auxentry);
    return 0;
}
```

## key 删除 1：dictDelete

- 删除 key，当 key 不存在时，返回 DICT_ERR

```c
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}
```

## key 删除 2：dictUnlink

- 删除 key 对应节点，当 key 不存在时，返回 DICT_ERR
- 当 key 存在时，删除节点，返回 entry；但不做 free 操作，需要 client 控制 free；配合 dictFreeUnlinkedEntry 方法使用

```c
dictEntry *dictUnlink(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,1);
}

// Search and remove an element
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    // 空hashtable检查
    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    // 协助rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 计算hash
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {

                // 链表节点删除
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;

                // 根据入参做 free 操作
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }

                d->ht[table].used--;
                return he;
            }
            prevHe = he;
            he = he->next;
        }
        // 不在 rehash 过程中，只需要操作ht[0]
        if (!dictIsRehashing(d)) break;
    }

    // 未找到
    return NULL;
}
```

## 节点 free ：dictFreeUnlinkedEntry

```c
void dictFreeUnlinkedEntry(dict *d, dictEntry *he) {
    if (he == NULL) return;
    dictFreeKey(d, he);
    dictFreeVal(d, he);
    zfree(he);
}
```

## dict clean：dictRelease

```c
void dictRelease(dict *d)
{
    // hashtable 节点及空间释放
    _dictClear(d,&d->ht[0],NULL);
    _dictClear(d,&d->ht[1],NULL);

    // dict 空间释放
    zfree(d);
}

// hashtable clean
int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    unsigned long i;

    // 释放每一个节点
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        dictEntry *he, *nextHe;

        // 执行 hook 方法
        if (callback && (i & 65535) == 0) callback(d->privdata);

        if ((he = ht->table[i]) == NULL) continue;
        while(he) {
            nextHe = he->next;
            dictFreeKey(d, he);
            dictFreeVal(d, he);
            zfree(he);
            ht->used--;
            he = nextHe;
        }
    }

    // hashtable空间释放
    zfree(ht->table);

    // hashtable 字段初始化
    _dictReset(ht);

    return DICT_OK;
}
```

## key 查询：dictFind

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    // empty check
    if (dictSize(d) == 0) return NULL;

    // help rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // calculate hash
    h = dictHashKey(d, key);

    // find dictEntry
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

## key 查询 value：dictFetchValue

```c
void *dictFetchValue(dict *d, const void *key) {
    dictEntry *he;

    // find key entry
    he = dictFind(d,key);

    // return entry's value
    return he ? dictGetVal(he) : NULL;
}
```

## 最小化扩容：dictResize

```c
int dictResize(dict *d)
{
    unsigned long minimal;

    // 判断是否允许扩容、是否在rehash中
    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;

    // 获取最小使用空间
    minimal = d->ht[0].used;
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;

    // dictExpand 扩容
    return dictExpand(d, minimal);
}
```

## dict 扩容 - dictExpand

```c

int dictExpand(dict *d, unsigned long size)
{
    // 检查是否在rehash中，或者已存在节点大于hashtable大小
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    // 初始化新的hashtable
    dictht n;
    // 计算新hashtable大小
    unsigned long realsize = _dictNextPower(size);
    // 大小检查
    if (realsize == d->ht[0].size) return DICT_ERR;

    // 分配空间及初始化
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    // 检查hashtable首次分配空间
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    // 扩容完成，准备reHash
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```

## 获取迭代器：dictGetIterator

```c
dictIterator *dictGetIterator(dict *d)
{

    // 迭代器初始化
    dictIterator *iter = zmalloc(sizeof(*iter));

    iter->d = d;
    iter->table = 0;
    iter->index = -1;
    iter->safe = 0;
    iter->entry = NULL;
    iter->nextEntry = NULL;
    return iter;
}
```

## 获取 safe 迭代器：dictGetSafeIterator

```c
dictIterator *dictGetSafeIterator(dict *d) {

    // 迭代器初始化
    dictIterator *i = dictGetIterator(d);

    // 设置安全标志
    i->safe = 1;
    return i;
}
```

## 迭代器遍历：dictNext

- 完成两个 hashtable 的遍历
- 完成冲突节点的链表遍历

```c
dictEntry *dictNext(dictIterator *iter) {
    while (1) {
        if (iter->entry == NULL) {
            dictht *ht = &iter->d->ht[iter->table];

            // 第一次遍历
            if (iter->index == -1 && iter->table == 0) {
                if (iter->safe)
                    // 迭代器数量+1
                    iter->d->iterators++;
                else
                    // 记录dict指纹
                    iter->fingerprint = dictFingerprint(iter->d);
            }
            iter->index++;

            // 检查遍历索引大于 buckets 数量
            if (iter->index >= (long) ht->size) {
                // rehash 中检查，继续遍历ht[1]
                if (dictIsRehashing(iter->d) && iter->table == 0) {
                    iter->table++;
                    iter->index = 0;
                    ht = &iter->d->ht[1];
                } else {
                    break;
                }
            }
            iter->entry = ht->table[iter->index];
        } else {
            iter->entry = iter->nextEntry;
        }

        if (iter->entry) {
            // 设置 nextEntry
            iter->nextEntry = iter->entry->next;
            return iter->entry;
        }
    }
    return NULL;
}
```

## 迭代器删除：dictReleaseIterator

```c
void dictReleaseIterator(dictIterator *iter)
{
    // 检查迭代器是否使用
    if (!(iter->index == -1 && iter->table == 0)) {
        if (iter->safe)
            // 迭代器数量-1
            iter->d->iterators--;
        else
            // 检查指纹对比
            assert(iter->fingerprint == dictFingerprint(iter->d));
    }
    zfree(iter);
}
```

## 获取随机 key：dictGetRandomKey

```c
dictEntry *dictGetRandomKey(dict *d) {
    dictEntry *he, *orighe;
    unsigned long h;
    int listlen, listele;

    // empty check
    if (dictSize(d) == 0) return NULL;

    // help rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    if (dictIsRehashing(d)) {
        // 在 rehash 过程中时
        do {
            // 随机一个大于 rehashidx，小于 ht[1] 大小的值，找到非空 bucket
            h = d->rehashidx + (random() % (d->ht[0].size + d->ht[1].size - d->rehashidx));
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] : d->ht[0].table[h];
        } while(he == NULL);
    } else {
        // 不在 rehash 过程中时，在 ht[0] buckets 范围内随机找到一个非空节点
        do {
            h = random() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }

    listlen = 0;
    orighe = he;

    // 统计链表长度
    while(he) {
        he = he->next;
        listlen++;
    }
    // 随机一个小于链表长度的值
    listele = random() % listlen;
    he = orighe;

    // 定位这个值
    while(listele--) he = he->next;
    return he;
}
```

## 公平获取随机 key：dictGetFairRandomKey

```c

```

## 批量获取随机 key：dictGetSomeKeys

```c

```

## rehash

```c
int dictRehash(dict *d, int n) {
    // 最大空 bucket 检测次数
    int empty_visits = n*10;

    // rehash 中检查
    if (!dictIsRehashing(d)) return 0;

    // step 控制
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        // rehashidx overflow check
        assert(d->ht[0].size > (unsigned long)d->rehashidx);

        // 查找非空 bucket
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }

        // 拿到 entry
        de = d->ht[0].table[d->rehashidx];

        // 处理链表中的所有entry
        while(de) {
            uint64_t h;
            nextde = de->next;

            // 计算新的 bucket 位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;

            // 头节点插入
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }

        // 旧的 bucket 置空
        d->ht[0].table[d->rehashidx] = NULL;

        // rehash 位置+1
        d->rehashidx++;
    }

    // rehash 完成检查
    if (d->ht[0].used == 0) {
        // 释放旧的 hashtable
        zfree(d->ht[0].table);

        // 使用扩容后的 hashtable 替换旧的 hashtable
        d->ht[0] = d->ht[1];

        // ht[1] 重置
        _dictReset(&d->ht[1]);

        // rehash 标志重置
        d->rehashidx = -1;
        return 0;
    }

    return 1;
}
```
