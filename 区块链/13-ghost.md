---
layout: posts
title: "13-ghost"
date: 2025-01-10 21:12:12
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "eth"



---

简介 <!--more-->



eth的ghost协议。

eth 出块时间10多秒，不像btc10分钟。

都是在应用层的 共识协议，底层是个 p2p的协议。



两个miner 同时 获得记账权就会发生临时性分叉。eth 由于出块快，所以这样的临时性分叉就多。

btc 中的出块奖励其实只有最后保留下来的那条链上的保留了，其他分叉链的orphan block 孤儿块 奖励被作废了。



# ghost协议

下图是一个 粗糙的设计，不是实际执行的协议。 ![截屏2024-12-30 20.53.12](13-ghost/%E6%88%AA%E5%B1%8F2024-12-30%2020.53.12.png)

> 假如 一个 block挖到奖励 3个 eth ，那在orphan链上的 uncle block 也有奖励，而且主链上的block 打包uncle block的内容还能获得 额外的奖励，但是 主链上的 block 只能打包两个uncle block 。
> uncle block 获得了大量的好处，实际上鼓励了 分叉后及时合并，不要执拗的去竞争自己的那条链了。

![截屏2024-12-30 21.09.13](13-ghost/%E6%88%AA%E5%B1%8F2024-12-30%2021.09.13.png)

> uncle block 可以是 很久以前的block

![截屏2024-12-30 21.13.04](13-ghost/%E6%88%AA%E5%B1%8F2024-12-30%2021.13.04.png)

> 往前 7 代算uncle block，这样包括的 uncle block 数量有限负担小，还能因为 uncle 奖励越来越小，鼓励及时合并；注意 兄弟block 和 叔叔block 是不一样的，而且只有第一个叔叔block 有奖励，防fork attack。

uncle block 是可以得到静态奖励的，也就是那个 7/8 等，但是得不到动态奖励，像交易的gas fee ，同理 uncle block 中的 合约 不会被执行。

eth 没有 定期减半的 规定；



![截屏2024-12-30 21.51.06](13-ghost/%E6%88%AA%E5%B1%8F2024-12-30%2021.51.06.png)