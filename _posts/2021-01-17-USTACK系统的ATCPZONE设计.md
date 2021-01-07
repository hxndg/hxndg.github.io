---
layout:     post   				    # 使用的布局（不需要改）
title:      USTACK系统的ATCPZONE设计
subtitle:   不能不学习啊 #副标题
date:       2021-01-07 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# USTACK系统的ATCPZONE设计

## 前言
简单看看ATCP ZONE怎么做的

## 分类流程

分配流程有限从per cpu cache取出来。
+ atcp_zalloc函数，会将当前当前ATCP进程的ID保存到`atcpid`里
+ 每个ATCP线程都会首先尝试从自己的per atcp cache也就是一个长长的链表中取出来数据：这个链表每个元素的第一个字节位置存储的是下个元素的位置`cache->zc_items[j] = ((void **) item)[MM_NEXT_ADDR_POSITION];`。取成功就返回，取失败就找缓存池申请`atcp_zpool_alloc(j, zzone->zpool, flags | M_DOMAIN);`。然后再次尝试分配。
+ 缓存池申请函数`atcp_zpool_alloc`首先判断分配多少数量为佳，以`cache->zc_batchalloc`为上限。然后锁住每个`zpool`对应的锁，然后每次从per `zpool`的空闲数据链表里拆出来一个item，然后把新的item放到每个atcp cache的链表头部。如果满足直接释放锁返回，反之调用`atcp_zpool_slab`函数给zpool申请新元素
+ 函数`atcp_zpool_slab`函数就开始干脏活累活了，还是要锁住`ZP_LOCK(zpool, domain) (&(zpool)->zp_lock[domain])`，然后按照part_slab, free_slab的顺序尝试分配结构，成功的话，`while (slab->us_freecount > 0 && ZP_FREECNT(zpool, domain) <= zpool->zp_freemin)`通过位图计算出part_slab上的第一个空闲目标直接放到zpool上，失败就调用`atcp_slab_alloc`分配新的slab，直接插入到part_slab上。上面无论失败还是成功如果发现alloc一个slab不够满足`ZP_FREECNT(zpool, domain) < zpool->zp_freemin`都会继续循环。
+ 函数`atcp_slab_alloc`是向huge page申请内存的操作。这部分就不赘述了，下面是调用流程。

## ZTCP_ZONE的初始化

这部分主要看怎么计算出来的着色操作，




## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
