# HashTable

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

# Redis-HashTable-Interfaces

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

## 批量获取随机 key：dictGetSomeKeys

- 不保证返回指定数量的随机节点，也不保证不返回重复节点

```c
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count) {
    unsigned long j;
    unsigned long tables;
    unsigned long stored = 0, maxsizemask;
    unsigned long maxsteps;

    // 检查 count 是否大于 两个 hashtable 长度之和
    if (dictSize(d) < count) count = dictSize(d);

    // 初始化最大查找次数
    maxsteps = count*10;

    // 协助 rehash
    for (j = 0; j < count; j++) {
        if (dictIsRehashing(d))
            _dictRehashStep(d);
        else
            break;
    }

    // 确定 hashtable 查询数量范围
    tables = dictIsRehashing(d) ? 2 : 1;

    // 获取最大掩码
    maxsizemask = d->ht[0].sizemask;
    if (tables > 1 && maxsizemask < d->ht[1].sizemask)
        maxsizemask = d->ht[1].sizemask;

    // 计算 bucket 初始位置
    unsigned long i = random() & maxsizemask;
    // 连续空 bucket 统计
    unsigned long emptylen = 0;
    while(stored < count && maxsteps--) {
        for (j = 0; j < tables; j++) {

            // 如果在 rehash 中，在 ht[0] 中 0 - rehashidx - 1 的值已经完成迁移
            if (tables == 2 && j == 0 && i < (unsigned long) d->rehashidx) {
                // 检查初始位置有效性，如果越界，赋值为 rehashidx
                if (i >= d->ht[1].size)
                    i = d->rehashidx;
                else
                    // 在ht[1] 中继续
                    continue;
            }

            // index overflow 检查
            if (i >= d->ht[j].size) continue;
            dictEntry *he = d->ht[j].table[i];

            // 检查 empty bucket
            if (he == NULL) {
                emptylen++;
                // 如果连续5个空 bucket，并且大于目标 entry 数量，重新计算初始 bucket 位置
                if (emptylen >= 5 && emptylen > count) {
                    i = random() & maxsizemask;
                    emptylen = 0;
                }
            } else {
                emptylen = 0;
                // 将链表的所有节点放入结果集
                while (he) {
                    *des = he;
                    des++;
                    he = he->next;
                    stored++;
                    if (stored == count) return stored;
                }
            }
        }

        // 连续递增，并且防止越界
        i = (i+1) & maxsizemask;
    }
    return stored;
}
```

## 公平获取随机 key：dictGetFairRandomKey

```c
dictEntry *dictGetFairRandomKey(dict *d) {

    // 定义结果集，默认长度15
    dictEntry *entries[GETFAIR_NUM_ENTRIES];

    // 获取随机节点
    unsigned int count = dictGetSomeKeys(d,entries,GETFAIR_NUM_ENTRIES);

    // 结果集为空，则调用 dictGetRandomKey 返回一个随机节点
    if (count == 0) return dictGetRandomKey(d);

    // 根据结果集长度，随机返回一个节点
    unsigned int idx = rand() % count;
    return entries[idx];
}
```

## Debug 接口获取 htstats： dictGetStats

```c
void dictGetStats(char *buf, size_t bufsize, dict *d) {
    size_t l;
    char *orig_buf = buf;
    size_t orig_bufsize = bufsize;

    // 获取 ht[0] 状态信息
    l = _dictGetStatsHt(buf,bufsize,&d->ht[0],0);
    buf += l;
    bufsize -= l;

    // 获取 ht[1] 状态信息
    if (dictIsRehashing(d) && bufsize > 0) {
        _dictGetStatsHt(buf,bufsize,&d->ht[1],1);
    }

    // set '\0' 为 buf 尾元素
    if (orig_bufsize) orig_buf[orig_bufsize-1] = '\0';
}

// 获取 hashtable 状态
size_t _dictGetStatsHt(char *buf, size_t bufsize, dictht *ht, int tableid) {
    unsigned long i, slots = 0, chainlen, maxchainlen = 0;
    unsigned long totchainlen = 0;

    // 链表长度统计
    unsigned long clvector[DICT_STATS_VECTLEN];
    size_t l = 0;

    // empty check
    if (ht->used == 0) {
        return snprintf(buf,bufsize,"No stats available for empty dictionaries\n");
    }

    // 结果集初始化
    for (i = 0; i < DICT_STATS_VECTLEN; i++) clvector[i] = 0;
    for (i = 0; i < ht->size; i++) {
        dictEntry *he;

        // 空 bucket 检查
        if (ht->table[i] == NULL) {
            clvector[0]++;
            continue;
        }

        // 非空 slot 统计
        slots++;

        // 计算链表长度
        chainlen = 0;
        he = ht->table[i];
        while(he) {
            chainlen++;
            he = he->next;
        }

        // 根据链表长度统计 key 数量
        clvector[(chainlen < DICT_STATS_VECTLEN) ? chainlen : (DICT_STATS_VECTLEN-1)]++;

        // 计算最长链表
        if (chainlen > maxchainlen) maxchainlen = chainlen;

        // 统计链表长度和
        totchainlen += chainlen;
    }

    // 可读信息生成：hashtable 信息、size、使用数量、slots、最长链表，链表节点占比、使用率
    l += snprintf(buf+l,bufsize-l,
        "Hash table %d stats (%s):\n"
        " table size: %ld\n"
        " number of elements: %ld\n"
        " different slots: %ld\n"
        " max chain length: %ld\n"
        " avg chain length (counted): %.02f\n"
        " avg chain length (computed): %.02f\n"
        " Chain length distribution:\n",
        tableid, (tableid == 0) ? "main hash table" : "rehashing target",
        ht->size, ht->used, slots, maxchainlen,
        (float)totchainlen/slots, (float)ht->used/slots);

    for (i = 0; i < DICT_STATS_VECTLEN-1; i++) {
        if (clvector[i] == 0) continue;
        if (l >= bufsize) break;
        l += snprintf(buf+l,bufsize-l,
            "   %s%ld: %ld (%.02f%%)\n",
            (i == DICT_STATS_VECTLEN-1)?">= ":"",
            i, clvector[i], ((float)clvector[i]/ht->size)*100);
    }

    /* Unlike snprintf(), teturn the number of characters actually written. */
    if (bufsize) buf[bufsize-1] = '\0';
    return strlen(buf);
}
```

## siphash hash 生成函数 dictGenHashFunction

```c
uint64_t dictGenHashFunction(const void *key, int len) {
    return siphash(key,len,dict_hash_function_seed);
}
```

## siphash 大小写敏感 hash 生成函数 dictGenHashFunction

```c
uint64_t dictGenCaseHashFunction(const unsigned char *buf, int len) {
    return siphash_nocase(buf,len,dict_hash_function_seed);
}
```

## 清空 dict： dictEmpty

```c
void dictEmpty(dict *d, void(callback)(void*)) {

    // hashtable 节点及空间释放
    _dictClear(d,&d->ht[0],callback);
    _dictClear(d,&d->ht[1],callback);

    // 重置 rehashidx 及迭代器计数
    d->rehashidx = -1;
    d->iterators = 0;
}
```

## 启用 rehash：dictEnableResize

```c
void dictEnableResize(void) {
    dict_can_resize = 1;
}
```

## 禁止 rehash：dictEnableResize

```c
void dictDisableResize(void) {
    dict_can_resize = 0;
}
```

## 限制步数 rehash：dictRehash

- iterators 迭代器数量必须为 0

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

## 限时 rehash：dictRehashMilliseconds

```c
int dictRehashMilliseconds(dict *d, int ms) {
    // 迭代中，
    if (d->iterators > 0) return 0;

    // 获取 rehash 开始时间
    long long start = timeInMilliseconds();
    int rehashes = 0;

    // 执行100次 rehash
    while(dictRehash(d,100)) {
        rehashes += 100;

        // 超时检查
        if (timeInMilliseconds()-start > ms) break;
    }

    // 返回 rehash 次数
    return rehashes;
}

long long timeInMilliseconds(void) {
    struct timeval tv;
    gettimeofday(&tv,NULL);
    return (((long long)tv.tv_sec)*1000)+(tv.tv_usec/1000);
}
```

## 设置 hash 函数种子：dictSetHashFunctionSeed

```c
static uint8_t dict_hash_function_seed[16];

void dictSetHashFunctionSeed(uint8_t *seed) {
    memcpy(dict_hash_function_seed,seed,sizeof(dict_hash_function_seed));
}
```

## 获取 hash 函数种子：dictGetHashFunctionSeed

```c
uint8_t *dictGetHashFunctionSeed(void) {
    return dict_hash_function_seed;
}
```

## dict 扫描：dictScan

- dictScan() 用于遍历字典的所有元素
- 开始遍历那一刻的所有元素，只要不被删除，肯定能被遍历到，不管字典扩展还是缩小
- 可能会返回重复元素，但是已经把返回重复元素的可能性降到了最低

```c

/* 对 v 进行二进制逆序操作 http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel */
static unsigned long rev(unsigned long v) {
    unsigned long s = CHAR_BIT * sizeof(v); // bit size; must be power of 2
    unsigned long mask = ~0UL;
    while ((s >>= 1) > 0) {
        mask ^= (mask << s);
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}

/*
字典遍历
v： 要遍历的 bucket 索引
*/
unsigned long dictScan(dict *d, unsigned long v, dictScanFunction *fn, dictScanBucketFunction* bucketfn, void *privdata) {
    dictht *t0, *t1;
    const dictEntry *de, *next;
    unsigned long m0, m1;

    // empty check
    if (dictSize(d) == 0) return 0;

    // 迭代器数量+1，防止 rehash
    d->iterators++;

    // 如果不在 rehash 中
    if (!dictIsRehashing(d)) {
        t0 = &(d->ht[0]);
        m0 = t0->sizemask;

        // bucket callback
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];

        // bucket 链表遍历
        while (de) {
            next = de->next;
            fn(privdata, de);  // 链表节点 callback
            de = next;
        }

        v |= ~m0;  // 保留 v 的低 n 位数，其余位全置为 1

        // [二进制反转](http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel)
        v = rev(v);  // 将 v 的二进制位进行翻转，所以，v的低 n 位数成了高 n 位数，并且进行了翻转
        v++;         // 自增
        v = rev(v);  // 再次二进制翻转
    } else {
        // 在 rehash 过程中
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        // 确保 t0 长度小于 t1
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        m0 = t0->sizemask;
        m1 = t1->sizemask;

        // bucket callback
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];

        // bucket 链表遍历
        while (de) {
            next = de->next;
            fn(privdata, de);  // 链表节点 callback
            de = next;
        }

        // 大的 hashtable t1 遍历
        do {
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);  // bucket callback
            de = t1->table[v & m1];

            // bucket 链表遍历
            while (de) {
                next = de->next;
                fn(privdata, de);  // 链表节点 callback
                de = next;
            }

            v |= ~m1;     // 保留 v 的低 n 位数，其余位全置为 1
            v = rev(v);   // 将 v 的二进制位进行翻转，所以，v的低 n 位数成了高 n 位数，并且进行了翻转
            v++;          // 自增
            v = rev(v);   // 再次二进制翻转

        // 直到 v 的低 m1-m0 位到低 m1 位之间全部为 0 为止
        } while (v & (m0 ^ m1));
    }

    // 迭代器数量-1
    d->iterators--;

    return v;
}
```

## 计算 hash： dictGetHash

- 使用的 hash 函数为初始化 dict 时设置的 hash 函数

```c
uint64_t dictGetHash(dict *d, const void *key) {
    return dictHashKey(d, key);
}
```

## 根据指针和 hash 查询 entry： dictFindEntryRefByPtrAndHash

```c
dictEntry **dictFindEntryRefByPtrAndHash(dict *d, const void *oldptr, uint64_t hash) {
    dictEntry *he, **heref;
    unsigned long idx, table;

    // empty check
    if (dictSize(d) == 0) return NULL;
    for (table = 0; table <= 1; table++) {
        // 计算 slot
        idx = hash & d->ht[table].sizemask;
        heref = &d->ht[table].table[idx];
        he = *heref;

        // 链表节点查询
        while(he) {

            // 通过指针对比代替 key 对比
            if (oldptr==he->key)
                return heref;
            heref = &he->next;
            he = *heref;
        }

        // 非 rehash 过程中，不需要查询 ht[1]
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```
