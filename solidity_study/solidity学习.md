---
layout: posts
title: solidity学习
date: 2025-12-4 15:27:21
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "solidity"

---



solidity文档 `https://learnblockchain.cn/docs/solidity/`

# 前言

## 首先要明确几个概念

1、**mapping 声明**：

mapping 声明后的变量不是方法，而是会实实在在存一些东西的。但是它和指针又有很多不一样。

指针：标识符 = **真实存储的地址**

```c++
int x = 100;
int *p = &x;  // p 变量里存着 x 的地址（比如 0x1000）
printf("%p", p); // 可以打印出 0x1000
*p = 200;        // CPU 读取 p 的值（0x1000），再去写那个内存
```

- `p` 是一个**真实存在的变量**，占 8 字节内存
- 地址 `0x1000` 被**明确记录下来**

mapping：标识符 = **临时计算的输入**

```solidity 
mapping(address => uint256) balances;
balances[0xAlice] = 100;
```

- `0xAlice` **不会被写入 storage**
- EVM 用它和 slot 计算出一个哈希（如 `0xabc...def`）
- **只把 `100` 写入 `0xabc...def` 这个槽**
- 如果你不知道 `0xAlice`，就永远找不到这个 `100`

> 💡 **指针是“存地址”，mapping 是“用 key 算地址”**。



# 几个实例

## 选举

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
/// @title 委托投票
contract Ballot {
    // 这里声明了一个新的复合类型用于稍后的变量。
    // 该变量用来表示一个选民。
    struct Voter {
        uint weight; // 计票的权重
        bool voted;  // 若为真，代表该人已投票
        address delegate; // 被委托人
        uint vote;   // 投票提案的索引
    }

    // 提案的类型.
    struct Proposal {
        bytes32 name;   // 简称（最长32个字节）
        uint voteCount; // 得票数  
    }

    address public chairperson;

    // 这声明了一个状态变量，为每个可能的地址存储一个 `Voter`。
    mapping(address => Voter) public voters;

    // 一个 `Proposal` 结构类型的动态数组
    Proposal[] public proposals;

    /// 为 `proposalNames` 中的每个提案，创建一个新的（投票）表决 memory 表示 从临时内存中读取，就是 获取外部输入。 
    constructor(bytes32[] memory proposalNames) {  // 这里用 memory 临时用一下，但是proposals.push 会让内容保存到永久 storage中。 
        chairperson = msg.sender;                  // msg.sender 是调用合约者（消息发送者）自己的地址。
        voters[chairperson].weight = 1;

        //对于提供的每个提案名称，
        //创建一个新的 Proposal 对象并把它添加到数组的末尾。
        for (uint i = 0; i < proposalNames.length; i++) {
            // `Proposal({...})` 创建一个临时 Proposal 对象，
            // `proposals.push(...)` 将其添加到 `proposals` 的末尾
            proposals.push(Proposal({
                name: proposalNames[i],
                voteCount: 0
            }));
        }
    }

    // 授权 `voter` 对这个（投票）表决进行投票。
    // 只有 `chairperson` 可以调用该函数。
    function giveRightToVote(address voter) external {
        // 若 `require` 的第一个参数的计算结果为 `false`，
        // 则终止执行，撤销所有对状态和以太币余额的改动。
        // 在旧版的 EVM 中这曾经会消耗所有 gas，但现在不会了。
        // 使用 require 来检查函数是否被正确地调用，是一个好习惯。
        // 你也可以在 require 的第二个参数中提供一个对错误情况的解释。
        require(
            msg.sender == chairperson,
            "Only chairperson can give right to vote."
        );
        require(
            !voters[voter].voted,
            "The voter already voted."
        );
        require(voters[voter].weight == 0);  // 避免 重复授权 ，这样无意义。 
        voters[voter].weight = 1;
    }

    /// 把你的投票委托到投票者 `to`。
    function delegate(address to) external {
        // 传引用
        Voter storage sender = voters[msg.sender];  //storage → "修改永久状态"后续操作直接修改区块链状态；memory → "我只需要临时计算"
        require(sender.weight != 0, "You have no right to vote");
        require(!sender.voted, "You already voted.");

        require(to != msg.sender, "Self-delegation is disallowed.");

        // 委托是可以传递的，只要被委托者 `to` 也设置了委托。
        // 一般来说，这种循环委托是危险的。因为，如果传递的链条太长，
        // 则可能需消耗的gas要多于区块中剩余的（大于区块设置的gasLimit），
        // 这种情况下，委托不会被执行。
        // 而在另一些情况下，如果形成闭环，则会让合约完全卡住。
        while (voters[to].delegate != address(0)) {  // 找到 委托链的最后一个委托者，address(0)指的是 null类似于委托链的终点。
            to = voters[to].delegate;

            // 不允许闭环委托
            require(to != msg.sender, "Found loop in delegation.");
        }

        Voter storage delegate_ = voters[to];      // 最终委托者 （最终投票者）

        // Voters cannot delegate to accounts that cannot vote.
        require(delegate_.weight >= 1);

        // Since `sender` is a reference, this
        // modifies `voters[msg.sender]`.
        sender.voted = true;
        sender.delegate = to;

        if (delegate_.voted) {
            // 若被委托者已经投过票了，直接增加得票数
            proposals[delegate_.vote].voteCount += sender.weight;
        } else {
            // 若被委托者还没投票，增加委托者的权重
            delegate_.weight += sender.weight;
        }
    }

    /// 把你的票(包括委托给你的票)，
    /// 投给提案 `proposals[proposal].name`.
    function vote(uint proposal) external {
        Voter storage sender = voters[msg.sender];
        require(sender.weight != 0, "Has no right to vote");
        require(!sender.voted, "Already voted.");
        sender.voted = true;
        sender.vote = proposal;

        // 如果 `proposal` 超过了数组的范围，则会自动抛出异常，并恢复所有的改动

        proposals[proposal].voteCount += sender.weight;
    }

    /// @dev 结合之前所有的投票，计算出最终胜出的提案 ，view 只读不修改状态，免gas费。
    function winningProposal() public view  
            returns (uint winningProposal_)    
    {
        uint winningVoteCount = 0;
        for (uint p = 0; p < proposals.length; p++) {
            if (proposals[p].voteCount > winningVoteCount) {
                winningVoteCount = proposals[p].voteCount;
                winningProposal_ = p;
            }
        }
    }
    // 调用 winningProposal() 函数以获取提案数组中获胜者的索引，并以此返回获胜者的名称
    function winnerName() external view
            returns (bytes32 winnerName_)
    {
        winnerName_ = proposals[winningProposal()].name;
    }
}
```

## 普通竞拍

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract SimpleAuction {
    // 拍卖的参数。时间可以是绝对的 unix 时间戳（自 1970-01-01 起的秒数）或以秒为单位的时间段。
    address payable public beneficiary;  // 标记该地址能够接收资金
    uint public auctionEndTime;

    // 拍卖的当前状态。
    address public highestBidder;
    uint public highestBid;

    // 允许取回的先前出价
    mapping(address => uint) pendingReturns;

    // 在结束时设置为 true，禁止任何更改。
    // 默认初始化为 `false`。
    bool ended;

    // 变更触发的事件。 事件就是区块链的"广播系统"，广播内容为参数内容。
    event HighestBidIncreased(address bidder, uint amount);
    event AuctionEnded(address winner, uint amount);

    // Errors 用来定义失败

    // 三斜杠注释是所谓的 natspec 注释。
    // 当用户被要求确认交易时将显示它们，或者当显示错误时。

    /// 拍卖已经结束。
    error AuctionAlreadyEnded();  // revert 交易回滚​ gas 退回；
    /// 已经有更高或相等的出价。
    error BidNotHighEnough(uint highestBid);
    /// 拍卖尚未结束。
    error AuctionNotYetEnded();
    /// 函数 auctionEnd 已经被调用。
    error AuctionEndAlreadyCalled();

    /// 创建一个简单的拍卖，拍卖时间为 `biddingTime`秒，代表受益人地址 `beneficiaryAddress`。
    constructor(
        uint biddingTime,
        address payable beneficiaryAddress
    ) {
        beneficiary = beneficiaryAddress;
        auctionEndTime = block.timestamp + biddingTime;
    }

    /// 在拍卖中出价，出价的值与此交易一起发送。
    /// 该值仅在拍卖未获胜时退款。
    function bid() external payable {
        // 不需要参数，所有信息已经是交易的一部分。
        // 关键字 payable 是必需的，以便函数能够接收以太。

        // 如果拍卖时间已过，则撤销调用。
        if (block.timestamp > auctionEndTime)
            revert AuctionAlreadyEnded();

        // 如果出价不高，则将以太币退回（撤销语句将撤销此函数执行中的所有更改，包括它已接收以太币）。
        if (msg.value <= highestBid)
            revert BidNotHighEnough(highestBid);

        if (highestBid != 0) {
            // 通过简单使用 highestBidder.send(highestBid) 退回以太币是一个安全风险，因为它可能会执行一个不受信任的合约。
            // 让接收者自行提取他们的以太币总是更安全。
            pendingReturns[highestBidder] += highestBid;
        }
        highestBidder = msg.sender;
        highestBid = msg.value;
        emit HighestBidIncreased(msg.sender, msg.value);
    }

    /// 取回出价（当该出价已被超越）
    function withdraw() external returns (bool) {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // 将其设置为零很重要，因为接收者可以在 `send` 返回之前再次调用此函数作为接收调用的一部分。
            pendingReturns[msg.sender] = 0;

            // msg.sender 不是 `address payable` 类型，必须显式转换为 `payable(msg.sender)` 以便使用成员函数 `send()`。
            if (!payable(msg.sender).send(amount)) {
                // 这里不需要调用 throw，只需重置未付款
                pendingReturns[msg.sender] = amount;
                return false;
            }
        }
        return true;
    }

    /// 结束拍卖并将最高出价发送给受益人。
    function auctionEnd() external {
        // 这是一个好的指导原则，将与其他合约交互的函数（即它们调用函数或发送以太）结构化为三个阶段：
        // 1. 检查条件
        // 2. 执行操作（可能更改条件）
        // 3. 与其他合约交互
        // 如果这些阶段混合在一起，其他合约可能会回调当前合约并修改状态或导致效果（以太支付）被多次执行。
        // 如果内部调用的函数包括与外部合约的交互，它们也必须被视为与外部合约的交互。

        // 1. 条件
        if (block.timestamp < auctionEndTime)
            revert AuctionNotYetEnded();
        if (ended)
            revert AuctionEndAlreadyCalled();

        // 2. 生效
        ended = true;
        emit AuctionEnded(highestBidder, highestBid);

        // 3. 交互
        beneficiary.transfer(highestBid);
    }
}
```

## **"检查-生效-交互"**原则

check、effect、interaction。

如果让 interaction交互 发生在 effect状态变化前面，可能导致**重入攻击**。

**重入攻击**就是别人反复调用你的一个函数或者方法，实现攻击的方法。 例如：

```solidity
if (highestBid != 0) {
    // 改为直接转账（易受攻击的版本）
    (bool success, ) = highestBidder.call{value: highestBid}("");
    require(success, "Transfer failed");
}
highestBidder = msg.sender;
highestBid = msg.value;
emit HighestBidIncreased(msg.sender, msg.value);
```

interaction交互（向这个地址中转账先发生）

```solidity
(bool success, ) = highestBidder.call{value: highestBid}("");
```

effect状态后改变

```solidity
highestBidder = msg.sender;
highestBid = msg.value;
```

假如此时有一个 黑客合约，（在被别的合约call调用 收款时，会自动 运行receive函数，这是区块链规则），

```solidity

receive() external payable {
    // 在接收以太币时，重新调用 bid 函数，出价略高于当前最高价
    auction.bid{value: currentHighestBid + 1 wei}();
}
```

它先出价1eth，别人进价2eth，它就会以之前的最高价被调用`highestBidder.call{value: highestBid}("")`，它在收到1eth退款后，接了一个 重新调用 bid 函数的操作，而且出价仅仅比之前高一点点如1.001eth。由于之前的  effect状态改变还未发生，导致还是之前自己老的出价 1eth作为 `highestBid` ，自己这高一点点的 新竞价，能够通过 bid中 check 步骤 `if (msg.value <= highestBid)` 等等这些条件，然后 再度刷新自己之前 最高竞价退款的步骤`highestBidder.call{value: highestBid}("")`,再次收到 1eth的退款，而再次在 黑客合约中调用receive函数，再次收到 1eth的退款，实现递归，直到花光gas 或者在自己写的某个条件停下来。 

## 盲拍

**具有约束力且保密**：防止竞标者在赢得拍卖后不发送以太币的唯一方法是让他们在出价时一起发送，哈希实现保密。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract BlindAuction {
    struct Bid {
        bytes32 blindedBid;
        uint deposit;
    }

    address payable public beneficiary;
    uint public biddingEnd;
    uint public revealEnd;
    bool public ended;

    mapping(address => Bid[]) public bids;

    address public highestBidder;
    uint public highestBid;

    // 允许提取之前出价
    mapping(address => uint) pendingReturns;

    event AuctionEnded(address winner, uint highestBid);

    // Errors 用来定义失败

    /// 函数被调用得太早。
    /// 请在 `time` 再试一次。
    error TooEarly(uint time);
    /// 函数被调用得太晚。
    /// 不能在 `time` 之后调用。
    error TooLate(uint time);
    /// 函数 auctionEnd 已经被调用。
    error AuctionEndAlreadyCalled();

    // 修改器是一种方便的方式来验证输入函数。
    // `onlyBefore` 应用于下面的 `bid`：新的函数体是修改器的主体，其中 `_` 被旧函数体替换。
    // solidity 中的 modifier 方法 很像 python中的 @ 后面的装饰器啊，就是一个 对被修饰方法的闭包式的 封装。 
    modifier onlyBefore(uint time) {
        if (block.timestamp >= time) revert TooLate(time);
        _;
    }
    modifier onlyAfter(uint time) {
        if (block.timestamp <= time) revert TooEarly(time);
        _;
    }

    constructor(
        uint biddingTime,
        uint revealTime,
        address payable beneficiaryAddress
    ) {
        beneficiary = beneficiaryAddress;
        biddingEnd = block.timestamp + biddingTime;
        revealEnd = biddingEnd + revealTime;
    }

    /// 以 `blindedBid` = keccak256(abi.encodePacked(value, fake, secret)) 的方式提交一个盲出价，它是一个hash值。
    /// 发送的以太币仅在出价在揭示阶段被正确揭示时才会退还。
    /// 如果与出价一起发送的以太币至少为 "value" 且 "fake" 不为真，则出价有效。
    /// 将 "fake" 设置为真并发送不准确的金额是隐藏真实出价的方式，但仍然满足所需的存款。
    /// 相同地址可以提交多个出价。
    // onlyBefore(biddingEnd) 就是 solidity使用前文装饰器的写法。 
    function bid(bytes32 blindedBid)
        external
        payable
        onlyBefore(biddingEnd)
    {
        bids[msg.sender].push(Bid({
            blindedBid: blindedBid,
            deposit: msg.value
        }));
    }

    /// 用户自己来揭示盲出价。
    /// 将获得所有正确盲出的无效出价的退款，以及除了最高出价之外的所有出价。
    // 在 Solidity 中，函数参数有三种存储位置：
    // memory- 易失性内存（可修改）
    // storage- 永久存储（可修改）
    // calldata- 只读调用数据（不可修改）
    function reveal(
        uint[] calldata values,
        bool[] calldata fakes,
        bytes32[] calldata secrets
    )
        external
        onlyAfter(biddingEnd)
        onlyBefore(revealEnd)
    {
        uint length = bids[msg.sender].length;
        require(values.length == length);
        require(fakes.length == length);
        require(secrets.length == length);

        uint refund; // 退款
        for (uint i = 0; i < length; i++) {
            Bid storage bidToCheck = bids[msg.sender][i]; // 公示数据是链上要记录的。
            // 获取用户提交的揭示数据
            (uint value, bool fake, bytes32 secret) =
                    (values[i], fakes[i], secrets[i]);
            if (bidToCheck.blindedBid != keccak256(abi.encodePacked(value, fake, secret))) {
                // 出价未能正确披露
                // 不退还存款。
                continue;
            }
            refund += bidToCheck.deposit;
            // 如果是真实出价且押金>=出价金额
            if (!fake && bidToCheck.deposit >= value) {
                if (placeBid(msg.sender, value))
                    refund -= value;
            }
            // 使发送者无法重新取回相同的存款。如果没有这一步，用户再调用 reveal函数，又会退一遍，这一步的目的相当于撤销承诺。
            bidToCheck.blindedBid = bytes32(0);
        }
        payable(msg.sender).transfer(refund);
    }

    /// 提取被超出出价的出价。
    function withdraw() external {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // 将其设置为零是重要的，
            // 因为，作为接收调用的一部分，
            // 接收者可以在 `transfer` 返回之前重新调用该函数。（可查看上面关于“条件 -> 生效 -> 交互”的标注）
            pendingReturns[msg.sender] = 0;

            payable(msg.sender).transfer(amount);
        }
    }

    /// 结束拍卖并将最高出价发送给受益人。
    function auctionEnd()
        external
        onlyAfter(revealEnd)
    {
        if (ended) revert AuctionEndAlreadyCalled(); // ended 在一开始 默认是0 也就是 false 默认没有结束。
        emit AuctionEnded(highestBidder, highestBid);
        ended = true;                                // 锁住了。不能反复 给受益人打钱。
        beneficiary.transfer(highestBid);
    }

    // 这是一个“内部”函数，这意味着它只能从合约本身（或从派生合约）调用。
    function placeBid(address bidder, uint value) internal
            returns (bool success)
    {
        if (value <= highestBid) {
            return false;
        }
        if (highestBidder != address(0)) {
            // 退款给之前的最高出价者。
            pendingReturns[highestBidder] += highestBid;
        }
        highestBid = value;
        highestBidder = bidder;
        return true;
    }
}
```

## 远程购买
双方必须将物品价值的两倍放入合约作为托管。 一旦发生状况，以太币将被锁定在合约中，直到买方确认他们收到了物品。 之后，买方将获得价值（他们存款的一半），而卖方将获得三倍的价值（他们的存款加上价值）。 其背后的想法是双方都有动力来解决这种情况，否则他们的以太币将永远被锁定。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;
contract Purchase {
    uint public value;
    address payable public seller;
    address payable public buyer;

    enum State { Created, Locked, Release, Inactive } // 定义一个枚举类
    // 状态变量的默认值为第一个成员，`State.created`
    State public state;

    modifier condition(bool condition_) {
        require(condition_);
        _;
    }

    /// 只有买方可以调用此函数。
    error OnlyBuyer();
    /// 只有卖方可以调用此函数。
    error OnlySeller();
    /// 当前状态下无法调用该函数。
    error InvalidState();
    /// 提供的值必须是偶数。
    error ValueNotEven();

    modifier onlyBuyer() {
        if (msg.sender != buyer)
            revert OnlyBuyer();
        _;
    }

    modifier onlySeller() {
        if (msg.sender != seller)
            revert OnlySeller();
        _;
    }

    modifier inState(State state_) {
        if (state != state_)
            revert InvalidState();
        _;
    }

    event Aborted();
    event PurchaseConfirmed();
    event ItemReceived();
    event SellerRefunded();

    // 确保 `msg.value` 是一个偶数。
    // 如果是奇数，除法将截断。
    // 通过乘法检查它不是奇数。
    constructor() payable {
        seller = payable(msg.sender);
        value = msg.value / 2;
        if ((2 * value) != msg.value)
            revert ValueNotEven();
    }

    /// 中止购买并收回以太币。
    /// 只能由卖方在合约被锁定之前调用。
    function abort()
        external
        onlySeller
        inState(State.Created)
    {
        emit Aborted();
        state = State.Inactive;
        // 我们在这里直接使用转账。
        // 可用于防止重入，因为它是此函数中的最后一个调用，我们已经改变了状态。
        seller.transfer(address(this).balance);
    }

    /// 作为买方确认购买。
    /// 交易必须包括 `2 * value` 以太币。
    /// 以太币将在调用 confirmReceived 之前被锁定。
    function confirmPurchase()
        external
        inState(State.Created)
        condition(msg.value == (2 * value))
        payable
    {
        emit PurchaseConfirmed();
        buyer = payable(msg.sender);
        state = State.Locked;
    }

    /// 确认你（买方）收到了物品。
    /// 这将释放锁定的以太币。
    function confirmReceived()
        external
        onlyBuyer
        inState(State.Locked)
    {
        emit ItemReceived();
        // 首先改变状态是很重要的，
        // 否则，使用 `send` 调用的合约可以再次调用这里。
        state = State.Release;

        buyer.transfer(value);
    }

    /// 此函数退款给卖方，即退还卖方的锁定资金。
    function refundSeller()
        external
        onlySeller
        inState(State.Release)
    {
        emit SellerRefunded();
        // 首先改变状态是很重要的，
        // 否则，使用 `send` 调用的合约可以再次调用这里。
        state = State.Inactive;

        seller.transfer(3 * value);
    }
}
```

## 微支付通道

对于没有接触过区块链的人来说应该先看看下面这个通俗的解释来理解一下 【微支付通道】的意思；

### （一）通俗的例子（新人必看）

比喻场景：你去咖啡店喝咖啡（微支付通道）

你每天都去同一家咖啡店，点咖啡价格是 **10 元**。

但是——每次都用区块链支付太慢、太贵（Gas）。
 所以你和店主想出一个办法：**先开一个“记账本”**（通道），只在开始和结束时上链，中间的每一杯咖啡都离线结算。

------

#### 🪙 第一步：开通通道（上链）

你和咖啡店老板在区块链上签了一个“合约”：

> 「我先在合约里存 100 元，代表我未来最多能喝 10 杯咖啡。」

这笔 100 元是**押金**，写进区块链里。

------

#### ✍️ 第二步：离线签名（离链支付）

你喝第一杯咖啡。
 老板要你“签个字”表示「我承认这杯咖啡花了 10 元」。

于是你写了一张**签署的凭证**：

```
我（客户）同意支付老板 10 元。
签名：Yushao
```

> 这张“签名凭证”不需要上链，只存在你和老板之间。

喝第二杯时，你再写：

```
我同意支付老板 20 元。
签名：Yushao
```

表示：**截止目前**我欠他 20 元（累计的）。

> 如果要加上一些复杂的参数来组装出签名的内容

> ```markdown
> 我欠 老板张三（recipient）的 钱（amount）是 20 元，
> 凭证编号（nonce）是 3，
> 这份凭证属于 合同号 #AABBCC（contractAddress）
> 签名：Yushao
> ```

------

##### 🔐 “签署内容”到底是什么？

签署的不是“文字”，而是一段数据的哈希。
 但你可以把它理解成是对这句话签名：

> “如果这张单据（余额=20元）是我签的，那我确实授权支付这么多。”

老板收到后，他自己不能改成“余额=30元”，因为那会导致签名验证失败。

✅ **签名证明了你确实说过这句话。**

------

#### 💡 第三步：通道结算（上链一次）

当你喝完最后一杯、或者不想继续喝了，
 老板把你最后一张“签名凭证”拿去链上：

> 「链上合约，请根据这份签名支付我 50 元。」

合约验证签名确实是你的 → 给老板转 50 元 → 把剩下的 50 元还你。
 整个过程中，**只有两次上链**：

1. 开通通道（存钱）
2. 关闭通道（结算）

中间那 10 次离线签名都没花 Gas。

### （二）官网的例子

例子中使用加密签名，使以太币在同一当事人之间的重复转移变得安全、即时，并且没有交易费用。 

Alice想发送一些以太给Bob， 即Alice是发送方，Bob是接收方。

Alice 只需要在链下发送经过加密签名的信息 (例如通过电子邮件)给Bob，它类似于写支票。

Alice和Bob使用签名来授权交易，这在以太坊的智能合约中是可以实现的。 Alice将建立一个简单的智能合约，让她传输以太币，但她不会自己调用一个函数来启动付款， 而是让Bob来做，从而支付交易费用。

该合约将按以下方式运作：

- Alice部署了 ReceiverPays 合约，附加了足够的以太币来支付将要进行的付款。


- Alice通过用她的私钥签署一个消息来授权付款。


- Alice将经过加密签名的信息发送给Bob。该信息不需要保密（后面会解释），而且发送机制也不重要。


- Bob通过向智能合约发送签名的信息来索取他的付款，合约验证了信息的真实性，然后释放资金。

#### 创建签名

Alice不需要与以太坊网络交互来签署交易，这个过程是完全离线的。 我们将使用 [web3.js](https://github.com/web3/web3.js) 或 [MetaMask](https://metamask.io/) 在浏览器中签署信息。

```js
/// 先进行哈希运算使事情变得更容易
var hash = web3.utils.sha3("message to sign");
web3.eth.personal.sign(hash, web3.eth.defaultAccount, function () { console.log("Signed"); });
/// web3.eth.personal.sign 把信息的长度加到签名数据中。 （这里是 web3.eth.defaultAccount）对 hash 进行签名。personal.sign 是 JSON-RPC personal_sign 的封装。签名会返回一个 signature（通常在 callback 的参数中），这里示例只打印 “Signed”。
```

#### 签署内容

对于履行付款的合约，签署的信息必须包括：

> 1. 收件人的钱包地址。
> 2. 要转移的金额。
> 3. 重放攻击的保护。

重放攻击是指一个已签署的信息被重复使用，以获得对第二次交易的授权。 为了避免重放攻击，我们使用与以太坊交易本身相同的技术， 即所谓的nonce，它是一个账户发送的交易数量。 智能合约会检查一个nonce是否被多次使用。

另一种类型的重放攻击可能发生在所有者部署 `ReceiverPays` 合约时， 先进行了一些支付，然后销毁该合约。后来， 他们决定再次部署 `RecipientPays` 合约， 但新的合约不知道以前合约中使用的nonces，所以攻击者可以再次使用旧的信息。

#### 组装参数

既然我们已经确定了要在签名信息中包含哪些信息， 我们准备把信息放在一起，进行哈希运算，然后签名。 简单起见，我们把数据连接起来。 [ethereumjs-abi](https://github.com/ethereumjs/ethereumjs-abi) 库提供了一个名为 `soliditySHA3` 的函数， 模仿Solidity的 `keccak256` 函数应用于使用 `abi.encodePacked` 编码的参数的行为。 这里有一个JavaScript函数，为 `ReceiverPays` 的例子创建了适当的签名。

```solidity
// recipient， 是应该被支付的地址。
// amount，单位是 wei, 指定应该发送多少ether。
// nonce， 可以是任何唯一的数字，以防止重放攻击。
// contractAddress， 用于防止跨合约的重放攻击。
function signPayment(recipient, amount, nonce, contractAddress, callback) {
    var hash = "0x" + abi.soliditySHA3(
        ["address", "uint256", "uint256", "address"],
        [recipient, amount, nonce, contractAddress]
    ).toString("hex");

    web3.eth.personal.sign(hash, web3.eth.defaultAccount, callback);
}
```

> `function signPayment(recipient, amount, nonce, contractAddress, callback) {`
>  定义一个用于生成并签名“支付凭证”的函数，最后通过 `callback` 返回签名（或错误）。
>
> `var hash = "0x" + abi.soliditySHA3( ... ).toString("hex");`
>  这行是关键，分解如下：
>
> - `abi.soliditySHA3([...types...], [...values...])`：
>    作用是按 **Solidity 的 `abi.encodePacked` 编码规则** 将值打包，然后对打包后的字节串做 keccak256（solidity 中的 `keccak256(abi.encodePacked(...))`）。它模仿 Solidity 中的行为，确保 JS 端生成的哈希与 Solidity 合约内预期一致。
> - `.toString("hex")`：把 `abi.soliditySHA3` 返回的 Buffer 或二进制结果转为 hex 字符串（不带 `0x` 前缀）。`"0x" + ...`：在以太坊生态中，十六进制字符串通常带 `0x` 前缀，所以这里加上 `0x`，形成标准的 hex hash 字符串（例如 `0xabc123...`）。
> - `personal.sign` 会**在签名前再次加上 Ethereum 消息前缀**（`\x19Ethereum Signed Message:\n${length}`）并将 `length` 设置为你传入 `hash` 的长度（如果你传入的是 32 字节 hash，这个 length 是 32，签名的是前缀+32字节内容）。因此，签名者实际签的是 `prefixed(hash)`，不是“裸哈希”。这导致在合约里直接用 `ecrecover` 去验证 `keccak256(abi.encodePacked(...))` 的原始哈希 **不会**与 `personal.sign` 的签名直接对应，除非你在合约里也做同样的前缀处理（即用 `keccak256("\x19Ethereum Signed Message:\n32" + hash)`）来重建签名时的消息摘要。或者，你可以用客户端签 `signTypedData`（EIP-712）来获得可被合约更方便校验的结构化签名。
> - callback 参数通常是 `(err, signature) => { ... }`，签名字符串格式通常是 65 字节并编码为 hex（`r` + `s` + `v`）。

#### 在Solidity中恢复信息签名者

web3.js 产生的签名是 `r`, `s` 和 `v` 的拼接的， 所以第一步是把这些参数分开。您可以在客户端这样做， 但在智能合约内这样做意味着你只需要发送一个签名参数而不是三个。 将一个字节数组分割成它的组成部分是很麻烦的， 所以我们在 `splitSignature` 函数中使用 [inline assembly](https://docs.soliditylang.org/zh-cn/v0.8.23/assembly.html) 完成这项工作（本节末尾的完整合约中的第三个函数）。

#### 支付通道

就是前面写的通俗例子的账本，但是这里提示一下，这个账本是单向支付通道；

#### 结算相关代码

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
// 这将报告一个由于废弃的 selfdestruct 而产生的警告
contract ReceiverPays {
    // 这里有一个关键的逻辑一定要搞明白，就是这个合约的所有人，就是出钱建这个合同的人，也就是owner。
    // 而掉用这个收款合约的人才是收款人，掉用这个合约的一些方法来收款。
    // 期间的签名sign 都是 onwer 出钱的人来签名，收款人掉用方法，给出出钱人的签名来要钱。
    address owner = msg.sender;

    mapping(uint256 => bool) usedNonces;

    constructor() payable {}    //{}：空函数体，表示构造函数不执行额外逻辑

    // claim是索要的意思索赔。
    function claimPayment(uint256 amount, uint256 nonce, bytes memory signature) external {
        require(!usedNonces[nonce]);
        usedNonces[nonce] = true;   // 标记为已经使用。

        // 这将重新创建在客户端上签名的信息。
        bytes32 message = prefixed(keccak256(abi.encodePacked(msg.sender, amount, nonce, this)));  //打包好，hash计算，加前缀，this- 当前合约地址（防止跨合约重放）

        require(recoverSigner(message, signature) == owner);

        payable(msg.sender).transfer(amount); // 向掉用这个合约方法的人打钱。
    }

    /// 销毁合约并收回剩余的资金。
    function shutdown() external {
        require(msg.sender == owner);
        selfdestruct(payable(msg.sender));
    }

    /// 签名方法。
    function splitSignature(bytes memory sig)
        internal
        pure
        returns (uint8 v, bytes32 r, bytes32 s)
    {
        require(sig.length == 65);

        assembly {
            // 前32个字节，在长度前缀之后。
            r := mload(add(sig, 32))
            // 第二个32字节。
            s := mload(add(sig, 64))
            // 最后一个字节（下一个32字节的第一个字节）。
            v := byte(0, mload(add(sig, 96)))
        }

        return (v, r, s);
    }

    function recoverSigner(bytes32 message, bytes memory sig)
        internal
        pure
        returns (address)  // 返回这个签名的地址。
    {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(sig);

        return ecrecover(message, v, r, s);
    }

    /// 构建一个前缀哈希值，以模仿 eth_sign 的行为。
    function prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }
}
```

#### 支付相关代码

下面是JavaScript代码，用于对上一节中的信息进行加密签名：

每条信息包括以下信息：

> - 智能合约的地址，用于防止跨合约重放攻击。
> - 到目前为止，欠接收方的以太币的总金额。

```js
function constructPaymentMessage(contractAddress, amount) {
    return abi.soliditySHA3(
        ["address", "uint256"],
        [contractAddress, amount]
    );
}

function signMessage(message, callback) {
    web3.eth.personal.sign(
        "0x" + message.toString("hex"),
        web3.eth.defaultAccount,
        callback
    );
}

// contractAddress， 是用来防止跨合约的重放攻击。
// amount，单位是wei，指定了应该发送多少以太。
// 回调函数 `callback` 会在签名操作完成后被调用。回调函数通常有两个参数：
//     - `error`：如果签名过程中发生错误，`error` 会包含错误信息；如果没有错误，`error` 为 `null`。
//     - `signature`：生成的签名字符串。如果签名成功，`signature` 会包含签名结果。

function signPayment(contractAddress, amount, callback) {
    var message = constructPaymentMessage(contractAddress, amount);
    signMessage(message, callback);
}
```

支付相关代码

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;
// 这将报告一个由于废弃的 selfdestruct 而产生的警告
contract SimplePaymentChannel {
    address payable public sender;      // 发送付款的账户。
    address payable public recipient;   // 接收付款的账户。
    uint256 public expiration;  // 超时时间，以防接收者永不关闭支付通道。

    constructor (address payable recipientAddress, uint256 duration)
        payable
    {
        sender = payable(msg.sender);
        recipient = recipientAddress;
        expiration = block.timestamp + duration;
    }

    /// 接收者可以在任何时候通过提供发送者签名的金额来关闭通道，
    /// 接收者将获得该金额，其余部分将返回发送者。
    function close(uint256 amount, bytes memory signature) external {
        require(msg.sender == recipient);
        require(isValidSignature(amount, signature));

        recipient.transfer(amount);
        selfdestruct(sender);
    }

    /// 发送者可以在任何时候延长到期时间。
    function extend(uint256 newExpiration) external {
        require(msg.sender == sender);
        require(newExpiration > expiration);

        expiration = newExpiration;
    }

    /// 如果达到超时时间而接收者没有关闭通道，
    /// 那么以太就会被释放回给发送者。
    function claimTimeout() external {
        require(block.timestamp >= expiration);
        selfdestruct(sender);
    }
   function isValidSignature(uint256 amount, bytes memory signature)
        internal
        view
        returns (bool)
    {
        // this 关键字指的是当前合约的实例。它通常用于获取合约的地址或调用合约上的函数。
        // 将合约地址和金额打包成一个字节序列。
        bytes32 message = prefixed(keccak256(abi.encodePacked(this, amount)));

        // 检查签名是否来自付款方。
        return recoverSigner(message, signature) == sender;
    }

    /// 下面的所有功能是取自 '创建和验证签名' 的章节。

    function splitSignature(bytes memory sig)
        internal
        pure
        returns (uint8 v, bytes32 r, bytes32 s)
    {
        require(sig.length == 65);

        assembly {
            // 前32个字节，在长度前缀之后。
            r := mload(add(sig, 32))
            // 第二个32字节。
            s := mload(add(sig, 64))
            // 最后一个字节（下一个32字节的第一个字节）。
            v := byte(0, mload(add(sig, 96)))
        }

        return (v, r, s);
    }

    // 表示该函数不读取或修改合约的状态，只基于输入参数进行计算。
    function recoverSigner(bytes32 message, bytes memory sig)
        internal
        pure
        returns (address)
    {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(sig);

        // 使用 `ecrecover` 函数从消息哈希 `message` 和签名的三个部分 `v`、`r`、`s` 中恢复签署者的地址。之前利用massage来做的签名中会包含 sender 信息，通过 ecrecover 方法，利用 message 和 签名还原出 sender 的地址；
        return ecrecover(message, v, r, s);
    }

    /// 构建一个前缀哈希值，以模仿eth_sign的行为。
    function prefixed(bytes32 hash) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", hash));
    }
}
```

#### 验证付款

这是以太坊智能合约和前端交互中常见的签名验证逻辑。

这意味着接收者对每条信息进行自行验证是至关重要的。 否则就不能保证接收者最终能够得到付款。

接收者应使用以下程序验证每条信息：

1. 验证签名信息中的合约地址是否与支付通道相符。
2. 验证新的总额是否为预期的数额。
3. 确认新的总额不超过代管的以太币数额。
4. 验证签名是否有效，是否来自于支付通道的发送方。

```solidity
// 这模拟了eth_sign 的JSON-RPC构建前缀的方法。
// 对原始消息哈希 hash 添加以太坊签名标准前缀，并重新哈希。
// 得到一个 最终被签名的消息哈希，这个才是 ecrecover 能正确恢复公钥所必需的值。
function prefixed(hash) {
    return ethereumjs.ABI.soliditySHA3(
        ["string", "bytes32"],
        ["\x19Ethereum Signed Message:\n32", hash]
    );
}

function recoverSigner(message, signature) {
    var split = ethereumjs.Util.fromRpcSig(signature);
    // 根据签名和消息哈希，恢复出公钥（不是私钥！）。
    var publicKey = ethereumjs.Util.ecrecover(message, split.v, split.r, split.s);
    // 将公钥（65字节） 转换为以太坊地址：
    // 对公钥做 Keccak256 哈希
    // 取最后 20 字节（即地址），.toString("hex") → 转为十六进制字符串
    var signer = ethereumjs.Util.pubToAddress(publicKey).toString("hex");
    return signer;
}

function isValidSignature(contractAddress, amount, signature, expectedSigner) {
    var message = prefixed(constructPaymentMessage(contractAddress, amount));
    var signer = recoverSigner(message, signature);
    return signer.toLowerCase() ==
        ethereumjs.Util.stripHexPrefix(expectedSigner).toLowerCase();
}
```

#### 模块化降低复杂性

> **“用模块化的方法来构建您的合约，可以帮助减少复杂性，提高可读性…”**

- 智能合约一旦部署就难以修改，安全至关重要。
- 如果把所有逻辑（转账、权限、业务规则等）都写在一个合约里，代码会非常臃肿，逻辑交织，极易出错。
- **模块化**：将不同功能拆分成独立的单元（比如用 Solidity 的 `library` 或单独的合约），每个单元只负责一个明确的任务。

**关注“模块间交互”，而非“所有函数的任意组合”: **

- 每个模块内部是“封闭”的（比如 `Balances` 库只管理余额）
- 你只需关注：**模块之间如何调用？传递什么参数？是否满足前置/后置条件？**
- 不需要担心模块内部的实现细节会意外影响其他部分。

想象你在造一辆车：

- **非模块化**：把引擎、刹车、方向盘全部焊死在一起，改一个零件可能影响所有功能。
- **模块化**：引擎是一个独立模块，刹车是另一个。你只需确保“引擎输出动力”、“刹车能减速”，它们之间通过标准接口（比如油门踏板）交互。
  → 即使引擎坏了，也不会导致刹车失灵。

在智能合约中，`Balances` 库就是“刹车系统”——独立、可靠、职责清晰。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.5.0 <0.9.0;

// 这个库保证了两个关键不变量（Invariants）：
// 不会出现负余额
// → 因为 move 中有 require(_balances[from] >= amount)
// 总余额守恒
// → 因为 -= amount 和 += amount 是原子操作，总和不变（假设初始总和正确）
// ✅ 一旦你验证了这个库的逻辑是正确的，在整个项目中都可以安全地使用它，而无需每次转账都重新思考“会不会溢出？会不会负数？”
library Balances {
    function move(mapping(address => uint256) storage balances, address from, address to, uint amount) internal {
        require(balances[from] >= amount);
        require(balances[to] + amount >= balances[to]);
        balances[from] -= amount;
        balances[to] += amount;
    }
}

// 这是一个简化版本的 ERC20 代币合约；
contract Token {
    // 声明 —— 声明了一种“通过键访问值”的存储模式。注意 mapping 不是一个方法。
    // solidity中的 mapping声明的这个变量 是“数据”（状态的组织方式），
    // 而方法（function）是“行为”（对数据的操作逻辑）。
    mapping(address => uint256) balances;
    // 方便直接引用Balance库中的方法；
    using Balances for *;
    // “两级哈希”，但强调：它是一次性计算，不是分步跳转。
    // 是一个接受两个键（address, address）的复合映射，其存储位置由这两个键共同哈希决定。
    // 像两把钥匙 开保险箱；而不是python中函数嵌套的闭包，也不是指针组成的链表；
    // allowed[所有者][被授权人] = 允许提取的最大代币数量
    mapping(address => mapping(address => uint256)) allowed;

    event Transfer(address from, address to, uint amount);
    event Approval(address owner, address spender, uint amount);

    function transfer(address to, uint amount) external returns (bool success) {
        balances.move(msg.sender, to, amount);
        emit Transfer(msg.sender, to, amount);
        return true;

    }

    // 这里是掉用别人授权的支付额度来进行支付； 
    // 用户可以授权别人代付（approve + transferFrom）
    // allowed[所有者地址][被授权人地址] = 允许提取的代币数量
    function transferFrom(address from, address to, uint amount) external returns (bool success) {
        require(allowed[from][msg.sender] >= amount);    // 在授权额度余额内；
        allowed[from][msg.sender] -= amount;             // 先改变状态（减少额度），再交互（转账）
        balances.move(from, to, amount);                 // 交互
        emit Transfer(from, to, amount);
        return true;
    }

    function approve(address spender, uint tokens) external returns (bool success) {
        require(allowed[msg.sender][spender] == 0, "");  // 只允许在当前授权额度为 0 时设置新额度
        allowed[msg.sender][spender] = tokens;           // 最多允许spender转走tokens个代币；
        emit Approval(msg.sender, spender, tokens);
        return true;
    }

    function balanceOf(address tokenOwner) external view returns (uint balance) {
        return balances[tokenOwner];
    }
}
```



# 安装solidity编译器

推荐直接使用  docker 

```bash
# 先换源，或者直接指定docker中proxy代理；
docker pull ethereum/solc:stable

docker run ethereum/solc:stable solc --version
```

所谓编译，就是把源代码脚本编译成机器能够执行的二进制代码，而 java 、python 之类的语言，是 用一个解释器中转 一下，给java 虚拟机或者python虚拟机使用。

```bash
docker run -v /path/to/contracts:/sources ethereum/solc:stable solc --bin --abi /sources/MyContract.sol -o /sources/output
```

> - `-v /path/to/contracts:/sources`：将本地合约目录挂载到容器内的 `/sources` 路径。
> - `solc --bin --abi ...`：调用 `solc` 编译器，生成字节码（`--bin`）和 ABI（`--abi`）。
> - `-o /sources/output`：将输出文件写入挂载目录中的 `output` 子目录（需要提前创建或确保路径可写）。



# 合约结构

## 状态变量

状态变量是指其值被永久地存储在合约存储中的变量。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract SimpleStorage {
    uint storedData; // 状态变量
    // ...
}
```

## 函数

函数是代码的可执行单位。 通常在合约内定义函数，但它们也可以被定义在合约之外。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.1 <0.9.0;

contract SimpleAuction {
    function bid() public payable { // 函数
        // ...
    }
}

// 定义在合约之外的辅助函数
function helper(uint x) pure returns (uint) {
    return x * 2;
}
```

## 修饰器

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.22 <0.9.0;

contract Purchase {
    address public seller;

    modifier onlySeller() { // 修饰器
        require(
            msg.sender == seller,
            "Only seller can call this."
        );
        _;
    }

    function abort() public view onlySeller { // 修饰器的使用
        // ...
    }
}
```

## 事件

事件是能方便地调用以太坊虚拟机日志功能的接口。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.22;

event HighestBidIncreased(address bidder, uint amount); // 事件

contract SimpleAuction {
    function bid() public payable {
        // ...
        emit HighestBidIncreased(msg.sender, msg.value); // 触发事件
    }
}
```

**`event` 会消耗 gas**，但**比写入 storage（状态变量）便宜得多**。

## 错误

错误(类型)允许您为失败情况定义描述性的名称和数据。 错误(类型)可以在 回滚声明 中使用。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.4;

/// 没有足够的资金用于转账。要求 `requested`。
/// 但只有 `available` 可用。
error NotEnoughFunds(uint requested, uint available);

contract Token {
    mapping(address => uint) balances;
    function transfer(address to, uint amount) public {
        uint balance = balances[msg.sender];
        if (balance < amount)
            revert NotEnoughFunds(amount, balance);
        balances[msg.sender] -= amount;
        balances[to] += amount;
        // ...
    }
}
```

- **用途**：用于**中断交易并回滚状态更改**，同时可以**向调用者传递结构化错误信息**。

- **gas 消耗**：比 `require()` + 字符串更省 gas（推荐用于自定义错误）。

## 结构类型

结构类型是可以将几个变量分组的自定义类型

```solidity 
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract Ballot {
    struct Voter { // 结构
        uint weight;
        bool voted;
        address delegate;
        uint vote;
    }
}
```

## 枚举类型

枚举可用来创建由一定数量的’常量值’构成的自定义类型

在合约中声明一个枚举变量:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract Purchase {
    enum State { Created, Locked, Inactive } // 创建一个枚举类型
    State public currentState; // 声明一个枚举类型的公共状态变量
}
```

给枚举赋值: 0 1 2...

```solidity
constructor() {
    currentState = State.Created; // 初始化为 Created
}

function lock() public {
    require(currentState == State.Created, "Can only lock when created");
    currentState = State.Locked;
}
```

比较枚举值:

```solidity
if (currentState == State.Locked) {
    // 执行某些操作
}
```

外部调用（如前端）读取:

```solidity
// 因为 `currentState` 是 `public`，Solidity 会自动生成一个 getter 函数。前端（如 ethers.js）调用时，**返回的是一个数字（0, 1, 2）**：

const state = await purchaseContract.currentState(); // 返回 0、1 或 2
```



# 类型

Solidity 是一种静态类型语言，这意味着每个变量（状态变量和局部变量）都需要被指定类型。 Solidity 提供了几种基本类型，可以用来组合出复杂类型。

Solidity中不存在“未定义”或“空”值null的概念， 但新声明的变量总是有一个取决于其类型的 [默认值](https://docs.soliditylang.org/zh-cn/latest/control-structures.html#default-value)。 为了处理任何意外的值，您应该使用 [revert 函数](https://docs.soliditylang.org/zh-cn/latest/control-structures.html#assert-and-require) 来恢复整个事务， 或者返回一个带有第二个 `bool` 值的元组来表示成功。

> **`revert` 会立即终止当前交易，**
> **回滚所有状态更改（就像什么都没发生），**交易已花的 gas 不退**（但不再继续花）**
>
> 并向调用者返回一个错误信息。
> 这在 **区块链环境** 中非常重要，因为交易要么**完全成功**，要么**完全失败**（原子性）。

```solidity
// 初始值 对 bool 是 false，对 uint 是 0，对 address 是 address(0)
mapping(address => uint) public scores;

// 用 revert 中断交易（推荐用于“必须存在”的场景）
function getScoreStrict(address user) public view returns (uint) {
    uint s = scores[user];
    if (s == 0) {
        revert UserNotFound(user); // 自定义错误，中断调用
    }
    return s;
}

// 返回 (value, success) 元组（推荐用于“可选”场景） 
function tryGetScore(address user) public view returns (uint score, bool exists) {
    uint s = scores[user];
    if (s == 0) {
        return (0, false); // 明确表示“不存在”
    }
    return (s, true);
}
// -----------------------  调用  -----------------------
(uint score, bool ok) = tryGetScore(user);
if (ok) {
    // 使用 score
} else {
    // 用户不存在，安全处理
}
// -----------------------  调用  -----------------------

// 如果你的业务允许 0 积分，那就不能用 0 判断是否存在。可以改用：
// 这样即使积分是 0，也知道用户是真实存在的。
mapping(address => uint) public scores;
mapping(address => bool) public isRegistered; // 显式标记是否注册
function getScoreSafe(address user) public view returns (uint, bool) {
    if (!isRegistered[user]) {
        return (0, false);
    }
    return (scores[user], true);
}
```

## 类型值

### bool 类型： 

注意也会有【**短路求值（Short-Circuit Evaluation）**】

```solidity
a && b // ：如果 a 为 false，不计算 b（因为结果已经是 false）；
a || b //：如果 a 为 true，不计算 b（因为结果已经是 true）。
```

### 整型：

`int` / `uint`: 分别表示有符号和无符号【**只有 0 和正数**】的不同位数的整型变量。 关键字 `uint8` 到 `uint256` （无符号整型，从 8 位到 256 位）以及 `int8` 到 `int256`， 以 8 位为步长递增。 `uint` 和 `int` 分别是 `uint256` 和 `int256` 的别名。

因为 1 字节 = 8 位，EVM 基于字节对齐，整数类型的位数只能是：8, 16, 24, 32, ..., 256（每次 +8），例如，对于 `uint32`，这是 `0` 到 `2**32 - 1`；对于 `int32`，这是 `-2**31` 到 `2**31 - 1`。

> 有两种模式在这些类型上进行算术。“包装” 或 “不检查” 模式和 “检查” 模式。 默认情况下，算术总是 “检查” 模式的，uint32为例，这意味着如果一个操作的结果超出了该类型的值范围【2**32 - 1】， 调用将通过一个 [失败的断言](https://docs.soliditylang.org/zh-cn/latest/control-structures.html#assert-and-require) 而被恢复。 您可以用 `unchecked { ... }` 来转换到“未检查”模式。

| `type(int8).min`   | `int8` 的最小值 = **-128**       |
| ------------------ | -------------------------------- |
| `type(int8).max`   | `int8` 的最大值 = **127**        |
| `type(int256).min` | `int256` 的最小值 = **-2²⁵⁵**    |
| `type(int256).max` | `int256` 的最大值 = **2²⁵⁵ - 1** |

```solidity
int8 x = type(int8).min; // x = -128
int8 y = -x;             // 想得到 +128，但 int8 最大只能存 127！

// 因为：
// ------------ 在 unchecked 中： ------------
int8 x = type(int8).min; // x = -128 (0b1000_0000)
-x                       // 二进制取反+1 → 还是 0b1000_0000 → 仍然是 -128！
// ------------ 在 unchecked 中： ------------
// 所以：
unchecked {
    assert(-x == x); // ✅ 成立！因为 -(-128) 在 8 位下还是 -128
}
int8 y = 5;  // 这是十进制 5
```





### 位运算

位操作是在数字的二进制补码表示上进行的。 这意味着，例如 `~int256(0) == int256(-1)`。

### 移位运算

- `x << y` 等同于数学表达式 `x * 2**y`。
- `x >> y` 等同于数学表达式 `x / 2**y`，向负无穷远的方向取整。

