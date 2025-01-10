---
layout: posts
title: "11-ETH概述"
date: 2025-01-10 21:11:12
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "eth"





---

简介 <!--more-->



![截屏2024-12-10 15.35.30](11-ETH%E6%A6%82%E8%BF%B0/%E6%88%AA%E5%B1%8F2024-12-10%2015.35.30.png)

工作量证明 改 权益证明



# 智能合约 smart contract

![截屏2024-12-10 19.26.03](11-ETH%E6%A6%82%E8%BF%B0/%E6%88%AA%E5%B1%8F2024-12-10%2019.26.03.png)



# 账户

![截屏2024-12-17 19.51.17](11-ETH%E6%A6%82%E8%BF%B0/%E6%88%AA%E5%B1%8F2024-12-17%2019.51.17.png)

btc 没有一个基于账户的概念，说明币的来源后，转帐余下的钱要转回自己的另一个账户。如上图。

而 eth 使用了基于账户的模型， 

双花攻击double spending attack 是 花钱的一方花两次，重放攻击repay attack 是 收钱的一方 重复收钱两次。



## externally owned account 外部账户

balance 钱包余额

nonce 计数器



## smart contract account 合约账户

balance

nonce

合约账户不能主动发起交易

code

storage 存储状态





> btc 打一枪换一个账户，但是只能合约需要有一个稳定的账户，好追责，所以基于账户；

