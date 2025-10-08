---
layout: posts
title: 三明治攻击Sanwich Attack和MEV
date: 2025-10-8 15:35:12
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "sandwich attack"
- "MEV"

---



# 三明治攻击（Sandwich attack）

三明治攻击是区块链去中心化交易（DEX）中一种常见的 **前后夹击（front-run + back-run）** 的 MEV（最大可提取价值）行为。攻击者在看到一笔尚未上链的交易（通常是一个会造成价格变动的大额换币交易）后，先发一笔把价格推上去的交易（前置），等目标交易被执行后，再发一笔把价格拉回来的交易（后置）。这样攻击者以更高的价格买入、以更高的价格卖出或反之，从而赚取差价，目标交易承担了额外滑点（更差的成交价格）。



# 计算例子

<section>
  <h2>三明治攻击 — 数字示例（基于恒定乘积 AMM, 如 Uniswap）</h2>

  <h3>假设（初始池子）</h3>
  <ul>
    <li>代币对：<strong>USDC / X</strong></li>
    <li>USDC 储备 <code>x = 100,000</code></li>
    <li>X 代币储备 <code>y = 100,000</code></li>
    <li>常数 <code>k = x * y = 10,000,000,000</code></li>
    <li>Alice 准备用 <code>1000 USDC</code> 买 X。</li>
    <li>攻击者决定对 Alice 做“三明治”：先用 <code>1000 USDC</code> 前置买入（front-run），
        等 Alice 的交易执行后，再把自己买到的 X 卖回（back-run）。</li>
  </ul>

  <hr/>

  <h3>一、理想化（无手续费）计算步骤</h3>
  <p>AMM 遵循恒定乘积：<code>k = x * y</code>。交易中若输入一侧变化，则另一侧按 <code>k</code> 保持不变。</p>

  <h4>1) 攻击者先前置买入 1000 USDC</h4>
  <p>
    前置后 USDC 储备：<code>x1 = x0 + 1000 = 101000</code><br/>
    对应的新 X 储备：<code>y1 = k / x1 = 10,000,000,000 / 101000 ≈ 99009.900990</code><br/>
    攻击者买到的 X：
  </p>
  <pre><code>Δy_A = y0 - y1 ≈ 100000 - 99009.900990 ≈ 990.0990099009902</code></pre>

  <h4>2) Alice 在新的池子上以 1000 USDC 买入</h4>
  <p>
    交易后 USDC 储备：<code>x2 = x1 + 1000 = 102000</code><br/>
    对应的新 X 储备：<code>y2 = k / x2 = 10,000,000,000 / 102000 ≈ 98039.2156861755</code><br/>
    Alice 买到的 X：
  </p>
  <pre><code>Δy_Alice = y1 - y2 ≈ 99009.900990 - 98039.215686 ≈ 970.685304</code></pre>

  <h4>3) 攻击者后置卖出（把先前买到的 Δy_A 卖回）</h4>
  <p>
    卖入后 X 储备：<code>y3 = y2 + Δy_A ≈ 98039.2156861755 + 990.0990099009902 ≈ 99029.3146960765</code><br/>
    卖出后 USDC 储备：<code>x3 = k / y3 ≈ 10,000,000,000 / 99029.3146960765 ≈ 100.——（见下）</code>
  </p>
  <p>攻击者得到的 USDC（无手续费）为：</p>
  <pre><code>USDC_out = x2 - x3
≈ 102000 - (10,000,000,000 / 99029.3146960765)
≈ 1019.8000392079985</code></pre>

  <p><strong>攻击者净收益（无手续费，不计 gas）：</strong></p>
  <pre><code>profit ≈ 1019.8000392079985 - 1000 ≈ 19.8000392 USDC</code></pre>

  <hr/>

  <h3>二、带手续费（UniswapV2 风格，0.3%）的计算</h3>
  <p>在 UniswapV2 中常用做法是：对 <em>输入量</em> 扣手续费（例如 0.3%），实际进入池子的量为：</p>
  <pre><code>amount_in_with_fee = amount_in * (1 - fee)</code></pre>

  <p>交换输出的闭式解（输入为 USDC，输出为 X）为：</p>
  <pre><code>amount_out = amount_in_with_fee * y / (x + amount_in_with_fee)</code></pre>

  <h4>1) 攻击者前置买入 1000 USDC（带 0.3%）</h4>
  <pre><code>fee = 0.003
amount_in = 1000
amount_in_with_fee = 1000 * (1 - 0.003) = 997.0

amount_out_attacker = 997 * 100000 / (100000 + 997)
                    ≈ 987.158034397061  (攻擊者买到的 X)
</code></pre>

  <p>交易后池子变为：</p>
  <pre><code>x1 = 100000 + 997 = 100997.0
y1 = 100000 - 987.158034397061 ≈ 99012.84196560294
</code></pre>

  <h4>2) Alice 用 1000 USDC 在新的池子上买</h4>
  <pre><code>amount_in_with_fee_Alice = 1000 * (1 - 0.003) = 997.0
amount_out_alice = 997 * y1 / (x1 + 997)
                 = 997 * 99012.84196560294 / (100997 + 997)
                 ≈ 970.685304108 (数值与无手续费情形接近但略有差别)
new reserves after Alice:
x2 = x1 + 997 = 100997 + 997 = 101994.0
y2 = y1 - amount_out_alice ≈ 99012.84196560294 - 970.685304108 ≈ 98042.15666149494
</code></pre>

  <h4>注：</h4>
  <p>（上面 x2 的计算方式可根据是否把手续费也计入池子具体实现细节略有差异；通常在 UniswapV2，手续费会被留在池子中，等效于输入量变为 <code>amount_in_with_fee</code> 并加到储备中。）</p>

  <h4>3) 攻击者后置卖出（卖回自己先前买到的 ≈ 987.158034397061 X）</h4>
  <pre><code>sell_x = 987.158034397061
sell_in_with_fee = sell_x * (1 - 0.003) ≈ 984.196560293870

USDC_out = sell_in_with_fee * x2 / (y2 + sell_in_with_fee)
        ≈ 984.196560293870 * 100997.0 / (98042.15666149494 + 984.196560293870)
        ≈ 1013.7216192144434  (注意：此为示例近似值)
</code></pre>

  <p><strong>攻击者净收益（带 0.3% 手续费，但不计链上 gas）：</strong></p>
  <pre><code>profit ≈ 1013.7216192144434 - 1000 ≈ 13.7216192 USDC</code></pre>

  <hr/>

  <h3>三、公式推导（简要）</h3>
  <p>恒定乘积：</p>
  <pre><code>k = x * y</code></pre>

  <p>若输入有效量为 <code>a</code>（已扣手续费），交易后：</p>
  <pre><code>x' = x + a
y' = k / x'
amount_out = y - y' = y - k / (x + a) = (y(x+a)-xy)/(x + a) = (a * y) / (x + a)
</code></pre>


  <p>因此在实现时步骤为：</p>
  <ol>
    <li>从 <code>amount_in</code> 扣除手续费：<code>a = amount_in * (1 - fee)</code></li>
    <li>计算输出：<code>amount_out = a * y / (x + a)</code></li>
  </ol>

  <hr/>

  <h3>四、注意事项与总结</h3>
  <ul>
    <li>上面所有数值均为示例计算，实际池子行为会因协议实现（手续费如何分配、是否把手续费留在池中等）而有细微差别。</li>
    <li>实际是否能盈利还需扣除链上真实的 <strong>gas 费用</strong> 与为抢跑支付的 <strong>priority fee</strong>；这些往往显著影响最终收益。</li>
    <li>不同池子规模、不同手续费率、不同交易额会严重影响攻击者的利润与对被攻击者的影响。</li>
  </ul>

  <hr/>

  <p style="font-size:0.9em;color:#555;">

  </p>

</section>



# 最大可提取价值 MEV （Maximal Extractable Value）

区块链交易是如何被处理的？

当你用eth这样的权益证明时proof of stake ，区块链发生一笔交易，交易先进入内存储memory pool，而不是直接上链，验证者决定哪些交易打包上链；验证者一般会选价格最高的交易优先打包。

 前面说的 sanwich attack 就是 一个 抢先交易和一个尾随交易构成，也实现了一次 MEV；

很多去中心化交易所提供了一个滑点限制来防止被MEV杀伤；他让你自己设置一个最大价格变动容忍百分比。

## 单区块清算攻击

有一种single block liquidation attack（单区块清算攻击），这种情况下恶意行为者可以操纵这个区块内某种代币的价格， 通常用闪电贷瞬间介入百万美元，大量买入某个资产，从而触发链上价格预言机中借贷协议中的自动清算，一旦清算，攻击者又会把这些代币卖出，让价格恢复到正常水平，整个过程发生在一个区块中，导致普通用户或者协议参与者收到损失。

### 应对策略

为应对这样的 single block liquidation attack，很多difi项目使用时间加权平均价格TWAT，而不是仅仅利用链上现货价格，使用的是 几个区块一定时间内的平均价格，这样单个区块中就很难操纵；

 还有一些策略是private隐藏起来交易，直到交易被确认，让MEV机器人不能抢先交易，但是不能避免验证者自己搞MEV操作。

还有一种策略是公平排序， 粗暴的执行先到先执行的策略，根本上杜绝重新排序套利的机会。或者用密码学原理执行乱序。这个策略会降低difi市场的高效和稳定，因为大家没有意愿参与验证；

还有一种策略是 区块空间拍卖block space auctions，追求公开拍卖，竞价优先级，但是这样还是会偏向最有钱的交易者。

说到底 MEV 是难以避免的。





