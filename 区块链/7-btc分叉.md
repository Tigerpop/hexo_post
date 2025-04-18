---
layout: posts
title: "7-btc分叉"
date: 2025-01-10 21:11:10
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "btc"
- "分叉"


---

简介 <!--more-->



# state fork

就是 前面说的 两个人都挖到了。

forking attack 分叉攻击

deliberate forking 人为分叉



## protocd fork

协议发生改变，导致了一些人升级了，一些人不同意升级，一些人没来得及升级，会造成 protocd fork

### hard fork

增加了bct协议的一些新的特性，一些人不喜欢不认，像 block size limit =1M 这样规则的改变时，一些人不同意，1M=1000 000个字节byte，如果一个交易是250个byte ，那就是约4000个交易，10分钟来一个这样的 block，4000个交易 就会平坦在10分钟内，约一秒7个交易。错过了这一批，下次一等又是十分钟，反反复复。

有人 提出要把block 大小上限更新到 4M，大部分人都认可，少数人不认。![截屏2024-12-02 19.26.27](7-btc%E5%88%86%E5%8F%89/%E6%88%AA%E5%B1%8F2024-12-02%2019.26.27.png)

新结点挖到了大block，旧结点挖到了小的block。也有可能新结点挖到了小的区块。

但是新结点认旧结点的那条链，只是旧结点不认 有大块的 链，这样，就只会出现一次分叉，就是 硬分叉。



### soft fork

如果有人把 block 大小上限定义改成 1M 变 0.5M ，大部分人都认可，少数人不认。 

![截屏2024-12-02 19.41.47](7-btc%E5%88%86%E5%8F%89/%E6%88%AA%E5%B1%8F2024-12-02%2019.41.47.png)

新结点不认老结点有大的块，总把有大块的链视为非法，**而老结点又认新结点挖的小块，导致了老结点会自愿跟在新结点的区块后面，就导致了分叉总是反反复复的发生**，但是由于链不够长又被抛弃，这样的总是出现临时性的分叉但是没有永久性的分叉就是 软分叉。



### coinbase 软分叉例子

![截屏2024-12-02 19.51.52](7-btc%E5%88%86%E5%8F%89/%E6%88%AA%E5%B1%8F2024-12-02%2019.51.52.png)

nonce 4字节byte 一共是 2^8*4 = 2^32 个位置，我们加上 coinbase 的前8位 也就是 2^64 个位置，一共就是 2^96 个位置，来作为 挖矿难度计算的潜在位置。

 想要 只要一个人有多少钱，最简单的方法是从 UTXO 中看看他有多少钱，但是 只有全结点会在内存中维护 UTXO，轻结点没有办法验证一个人有多少钱，

因此有人提议，把这个 UTXO 的内容也组成一个 merkle tree，把这个 UTXO的 merkle tree 的根 hash 值写入  coinbase 中，而每一笔交易的coinbase 最后又会 一直hash 汇总算到 block header 中的 根hash 值。这样可以让轻结点也可以验证 对方说的 资产是否真实。

这条软件更新的规则就会导致典型的软分叉，因为新规则导致了新结点产生链不适用于旧结点的操作，而旧结点认可新结点的行为。



## 总结

**hard fork **：是必须要 所有的结点都更新规则，才能 避免永久性的分叉。

**sorf fork ：** 是只要 系统中有半数以上算力的结点更新的软件，系统就不会出现永久性的分叉。（老结点的块跟在后面会被抛弃，自然只有临时分叉不会永久分叉）



# 补充图片帮助理解

注意：
**新链如果有多条，我们 只针对具体的某一条进行分析。**

![44444](7-btc%E5%88%86%E5%8F%89/44444-5635342.jpg)
