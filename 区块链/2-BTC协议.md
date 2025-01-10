---
layout: posts
title: "2-BTC协议"
date: 2025-01-10 21:05:10
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "btc"


---

简介 <!--more-->



# 一个小型区块链

![截屏2024-11-12 20.37.55](2-BTC%E5%8D%8F%E8%AE%AE/%E6%88%AA%E5%B1%8F2024-11-12%2020.37.55.png)



两种 hash 指针，一种是 把区块连接起来，形成一个链表用的；另一种是 指向 前面 某个交易的，说明币的来源。

这样避免 双花攻击 double spend attack。



> 一笔交易，A向B转账。
>
> A就需要知道B的BTC地址，BTC的地址是 公钥求hash 算出来的，所以A需要知道B的公钥；
>
> 同时所有人都要知道 A的公钥，因为A要对自己的行为进行数字签名，数字签名就是用私钥加密，然后让所有人用自己的公钥解密验证。所以 大家要知道 A的公钥。



![截屏2024-11-12 21.02.23](2-BTC%E5%8D%8F%E8%AE%AE/%E6%88%AA%E5%B1%8F2024-11-12%2021.02.23.png)

区块之间的hash 指针 其实 只与 block header 有关系，对block header 计算hash，串成了链表；

头中包括：

version、hash of previous block header、merkle root hash 、用来验算是否在合理范围的target、随机数nonce；

体中包括：

transaction list 

> H(block header) <= target ， 就算合格，而 nonce 作为随机数 在 block header 中，会对 hash 计算产生影响，我们试多个nonce随机数，看哪一个能让hash 值落入 target范围内。



# distributed consensus 分布式共识

大家都记账 就需要有一个分布式的共识。

asychromous faulty： 异步错误，分布式系统中存在异步就无法在分布式中达成共识。



我们假设一下，如果是每个账户都有投票权，那我一个超级计算机就拼命 生成 公私钥对（就是账户），那我就能获得很多投票权，形成对投票结果的操纵。女巫攻击【sybil attack】

> 女巫攻击【sybil attack】



## 过程展示（重点）

1、H(block header) <= target ， 就算合格，而 nonce 作为随机数 在 block header 中，会对 hash 计算产生影响，我们试多个nonce随机数，看哪一个能让hash 值落入 target范围内。

如果某个节点 找到了符合要求的 nonce 值，我们就说 这个节点获得了 记账权。

所谓“记账权”就是区块链中，写入下一个区块的权利， 只有找到这个nonce 值的结点，才有资格去发布下一个区块。



2、发布下一个区块之后，别的结点还需要对这个区块进行合法性认证。

例如： target 有一个 nBits 来衡量 target 的难度是否足够...验证block header是否合法； 再验证一下 是不是每个交易都是合法的，一要有合法的签名signed，二以前没有被花过。

同时，还要求 新加入的区块 要在 longest valid chain 最长合法链上。

![截屏2024-11-12 22.02.15](2-BTC%E5%8D%8F%E8%AE%AE/%E6%88%AA%E5%B1%8F2024-11-12%2022.02.15.png)

> 以上截图就是 forking attack，如果两个结点 都同时抢到了 记账权，优先生成下一个区块的一方会 成为longest valid chain，因此会胜出，而下一个区块没来得及生成的一方，会被 orphan block孤儿区块。



3、btc获得记账权的结点，还将获得 **block reward**，铸币权。

btc 的来源只可能是 transaction 或者 block reward 。

mining 挖矿 的 miner 矿工 就是干的这个事情，猛算抢夺 block reward。