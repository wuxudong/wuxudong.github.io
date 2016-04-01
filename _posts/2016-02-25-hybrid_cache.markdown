---
layout: post
title:  "多级混合式缓存"
date:   2016-02-25 19:20:36
categories: tech
---


# 多级混合式缓存

## 需求：

最早碰到这个问题是在豌豆荚的时候，另一个项目组里有个检测升级的服务。  
这个api批量上传用户本地apk的package name以及版本号等信息，服务端将这些信息与信息库进行对比，检测是否有恶意或可升级的app。  
信息库保存着所有app的格式版本等信息。  
假设有1M的app，每个app数十K的元信息，信息库整体在百G的规模。  
信息库平均每秒需要被访问10K次/秒[假设访问频率] * 100[平均每次请求里的 批量apk数量] = 1M次/s。

信息库特点: 
- 高并发
- 读请求 >> 写请求
- 数据量大
- 不要求一致性，允许有短时间的信息不一致。

## 原始方案

一开始是有个大的Terracotta ehcache集群，保存着整个信息库，大部分情况下运行正常，但是当数据越来越大时，Terracotta ehcache的__垃圾回收__产生了致命的停顿。  
__商用版的 Terracotta ehcache集群__提供了额外的功能，可以减少垃圾回收产生的影响，但__收费不菲__。  
同事也通过拆分Terracotta ehcache集群为几个小的集群来弱化GC和内存占用的影响，但也都治标不治本。

## 可能的方案

* 迁移到redis平台

  redis本身是一个优秀的缓存平台，内存占用可控，也算比较容易扩展(虽然当时还没发布redis3.0)， 但性能却不够，不足以支持1M/s的访问量。  

* 混合式多级cache
  
  可以以redis作为一个集中式的存储，每个服务器再用本地内存cache作为一级缓存，当cache miss时，从redis中获取并更新。  
  当某个app信息被更新后，除了更新redis，还需要通过广播订阅[这里使用了redis的pub/sub]通知所有本地内存cache也进行更新。
  这个方案结合了redis和本地访问的优点，数据扩展方便，热点数据基本都在本地内存，访问速度也有保证。
  

## 实现

参照spring RedisCache，实现了HybridCache，实际上它只是个代理类。  
内部有两个成员: localCache, remoteCache.   
优先从 localCache 读取数据，miss则从remoteCache获取，并更新localCache。
发生数据更新时，先更新localCache和remoteCache，并向reids发送广播，通知其他实例数据失效。


## 后续

用redis实现jsr的cache规范有个很大的缺点，cache规范里有cacheRegion的概念，不同CacheRegion不会互相影响，可以针对性地清空某个cacheRegion，这个在redis里实现起来特别低效。

spring-data-redis里对此的实现是，每个cacheRegion有个对应的list， 保存了这个cacheRegion中所有的key，当要清空这个cacheRegion时，就遍历这个list，一个个删除。

清空cacheRegion是个低频，很少见的操作。 但这个list的消耗不是一般的高，一个包含了所有key的巨大的list内存消耗很大。

更合理的做法是，如果没有这个需求，就不要实现cacheRegion，直接对clear跑出UnSupportedException.

或者为不同的CacheRegion配置不同的CacheManager，每个指向不同的redis实例，不过配置会麻烦一些。

## 其他

后来发现oschina出品了一个J2Cache，和上面这个基本是类似的，只是它另起了一套接口，而不是实现spring的Cache接口，因此不能使用spring的cache annotation，这点个人并不喜欢。











        