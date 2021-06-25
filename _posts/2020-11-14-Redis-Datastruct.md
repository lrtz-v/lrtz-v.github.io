---
title: Redis数据结构
date: 2020-11-14 20:30:01
tags: [Redis]
description: Redis数据结构简介
---

# Redis 数据结构

| 高级数据结构 | 底层数据结构                   |
| ------------ | ------------------------------ |
| String       | int、SDS                       |
| List         | zipList、linkedList、quickList |
| Hash         | zipList、hashTable             |
| Set          | int set、hashTable             |
| ZSet         | skiplist+hashTable             |
| GEO          | geo hash                       |
| HyperLogLog  |                                |
| Bitmap       |                                |
