---
layout: posts
title: solidity学习
date: 2025-10-4 10:27:21
description: "这是文章开头，显示在主页面，详情请点击此处。"
categories: 
- "区块链"
tags:
- "solidity"

---



solidity文档 `https://learnblockchain.cn/docs/solidity/`

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
