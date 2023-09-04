>**了解智能合约审计和最佳实践请**[点击查看文档](https://safful.com/) 

# Uniswap V4 Hook 最佳安全实践

近期 Uniswap Lab 官宣了下一代 AMM Uniswap V4 的开发进展，并公开了白皮书和代码仓库。这次 V4 的白皮书仅仅只有 3 页，原因是 V4 并没有对 AMM 的核心算法逻辑做太多修改，而是在 V3 的基础上，增加了一些新的特性，以满足更多的场景需求。 Safful 将基于目前已开源的代码，来看看 V4 带来了哪些新的特性，并针对 V4 推出的重要特性 Hook 分析最佳应用实践。

## 一、V4 与 V3 的差别

### 1.1 AMM

在 AMM 算法层面，Uniswap V4 并没有对 V3 做修改，依然使用基于恒定乘积 x\*y=k 的流动性算法。
在 Uniswap V3，每一个交易对可以有 4 个池子（原本是 3 个，后来增加了一个新的 1 bp pool），分别代表 0.01% , 0.05% , 0.3% , 1% 费率的池子，这些池子对应的 tick space 也各不相同，在创建池子的时候，只能选择这 4 种中的任一个。
在 Uniswap V4，每一个交易对理论上可以有任意数量的 pool，并且每一个 pool 的 fee rate 也可以是任意值，这些 pool 的 tick space 也可以是任意值。
这同时也带来了一个问题：Uniswap V4 中交易对的流动性将被碎片化，因此需要一个更有效的 router/aggregator 来帮助用户找到最优的交易路径。

### 1.2 Hooks

Hooks 是一组由第三方或者 Uniswap 官方开发的合约，在创建 pool 的时候，pool 可以选择绑定一个 hook. 之后在交易的特定阶段，pool 都会自动调用与之绑定的 Hook 合约。Uniswap V4 一共定义了这些可以执行 hook 合约代码的阶段：

- beforeInitialize
- afterInitialize
- beforeModifyPosition
- afterModifyPosition
- beforeSwap
- afterSwap
- beforeDonate
- afterDonate

分别表示在初始化 pool，添加 / 移除流动性，交易，捐赠等操作的前后，都可以调用 hook 合约。
Hook 合约需要显式指定在上述的哪些阶段进行执行，而 pool 则需要知道对应的 Hook 在某个阶段是否需要执行，为了节省 gas，这些 flag 都没有在合约中进行存储，而是需要 Hook 使用特定的地址来标明。具体判断的代码如下：

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021571318-5bab1836-7b47-45ae-9a83-e504c9f6e063.jpeg#averageHue=%23373a32&clientId=u4de60ac9-9e49-4&from=paste&id=u02832d8e&originHeight=209&originWidth=649&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u5b6ae055-1c1e-4fc0-8dc7-648867cabe6&title=)

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572091-60fdfa94-7d21-43a9-a713-2da55b05c3ce.jpeg#averageHue=%23424034&clientId=u4de60ac9-9e49-4&from=paste&id=u443405dc&originHeight=758&originWidth=821&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=ufa62d9c2-eab6-4709-8f09-04635187591&title=)

可以看出，Hook 地址的前 8 bit 都别用来标记在特定阶段此 Hook 是否需要执行的 flag。
因此，Hook 的开发者需要在部署合约的时候，产生出满足 Pool 要求的地址，这通常需要使用 Create 2 + 计算随机 Salt 来实现。
以下是白皮书中关于 Hook 执行的一个例子：

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572135-21ca9ad8-4b58-4a6a-81ef-5cf10f81199f.jpeg#averageHue=%23f9f9f9&clientId=u4de60ac9-9e49-4&from=paste&id=u169a5fd9&originHeight=578&originWidth=667&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u55b4f142-9e70-4f1b-81d7-dd383202dd2&title=)

可以看到在执行 swap 的前后，pool 会先检查 pool 对应的 Hook 是否开启了相应的 flag，如果开启了，就会自动调用 Hook 合约的相应函数。

### 1.3 动态 fee ratio

除了可以在特定阶段执行代码之外，Hook 还可以决定某一个 pool 的 swap fee 费率，和 withdraw 费率。withdraw 费率指的是用户在移除流动性时需要向 Hook 支付的费率。除此之外，Hook 还可以指定在 swap fee 中抽成一部分给自己。
在创建 pool 时，需要使用 fee 参数（uint 24）前 4 个 bit 来标记此 pool 是否使用动态 fee，以及是否启动 hook swap fee 和 withdraw fee：

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572172-b17cff8d-4cd8-4baf-b8d6-75a10ff20851.jpeg#averageHue=%233f3d34&clientId=u4de60ac9-9e49-4&from=paste&id=u11d185ba&originHeight=413&originWidth=744&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u20270e33-0dbf-4ebf-83fe-86b342af1f4&title=)

如果启动了动态 fee，那么 pool 会在每次 swap 之前，调用 Hook 合约来获取当前的 swap fee ratio. Hook 合约需要实现 getFee() 函数，返回当前的 swap fee ratio。
Hooks 让 Uniswap V4 成为了一个开发者平台，给了 AMM 更多的可能性。一些可以用 Hooks 来实现的功能包括 TWAMM（时间加权自动做市商）、Limit Order （限价订单）、LP 复投等将在后续的章节中详细介绍。

### 1.4 Singleton 合约

Uniswap V3 中每次创建新的 pool 都需要部署一个新的合约，这会消耗大量 gas，但是其实这些 pool 使用的代码是相同的，只是初始化参数不相同而已。Uniswap V4 引入了 Singleton 合约，用来管理所有的 pool，这样创建新的 pool 不再需要部署新的合约了，节省了部署合约的 gas。
另外，使用 Singleton 合约的好处是，可以减少交易过程中 token 的转账，因为所有的 pool 都在同一个合约中，所以可以直接在合约内部完成跨 pool 的 swap，而在 V3 中，跨 pool 的 swap 会需要将 token 在不同 pool 中转来转去，这会增加 gas。
同时，在 V4 中，所有 pool 使用同一个合约，并且合约内部的 token 记账也被简化为每种 token 按 token 来记账，而不是按 pool 来记账，这样一来如果想使用闪电贷借大量 token 也会更方便。

### 1.5 extload

为了方便 Hook 和其他合约的 integration，V4 合约增加了 extload 函数，这样合约所有内部 states 都变成外部可读了，所有 pool 的状态将对外部完全透明。

### 1.6 Flash Accounting

为了减少跨 pool swap 的 token 转账，V4 同时使用被称为 Flash Accounting 的方法，将 swap, add/remove liquidity/flash loan 的过程都标准化成一种类似闪电贷的过程：
（1）用户获取一个 lock
（2）用户进行任何操作，例如在多个 pool 中 swap，add/remove liquidity，或者通过闪电贷向 pool 借 token
（3）用户所有操作所产生的 token 转账都会被记录在 lock 中
（4）所有操作结束后，用户可以取走他获得的 token，同时需要支付 lock 中记录他需要支付的 token.
这些过程需要发生在一个交易中。
这样一来，如果一个交易中需要跨多个 pool 进行 swap，在结算时只需要两笔转账就够了。例如，在一次 ETH->USDC-BTC 这样的 swap 中，USDC 作为中间 token 完全不需要任何转账。

### 1.7 ERC 1155 mint/burn

Flash Accounting 可以减少同一笔交易中 swap 的 token 转账，通过使用 ERC 1155 token ，可以进一步减少多个交易的 token 转账。
V4 允许通过 ERC 1155 mint，将属于你的 token 保存在 V4 合约中，这样你就可以在多个交易中使用这些 token，而不需要每次都将 token 转账到 V4 合约中。
使用 ERC 1155 burn 可以将保存在 V4 合约中的 token 取出。
ERC 1155 适合频繁 swap 或 add/remove liquidity 的用户，这些用户可以将常用的 token 直接保存在 V4 合约中，这样可以减少 token 转账的 gas 开销。

## 二、Hooks 的最佳实践举例

### 2.1 TWAMM（时间加权自动做市商）

Alice 想在区块链上购买价值 1 亿美元的以太币。在现有的自动做市商（AMM）平台（例如 Uniswap）上执行这么大规模的订单将非常昂贵，因为这些平台可能会向 Alice 收取高昂的费用，以防止她利用内幕消息获取更好的价格。
为了获得更好的价格，Alice 的最佳选择是手动将订单拆分成几个较小的子订单，并在几个小时内逐步执行它们。这样做的目的是让市场有足够的时间来意识到她没有内幕消息，从而给予她更好的价格。但是，即使她发送几个较大的子订单，每个子订单仍然会对价格产生重大影响，同时还容易受到敌对交易者的「三明治攻击」。
TWAMM 通过代表 Alice 进行交易来解决这个问题。它将她的订单分解为无限多个微小的虚拟订单，以确保在时间上平滑地执行。同时，TWAMM 利用嵌入式 AMM 协议的特殊数学关系，能够在这些虚拟订单中分摊 Gas 成本。由于 TWAMM 在区块之间处理交易，因此也不容易受到「三明治攻击」。
总的来说，TWAMM 为 Alice 提供了一种更高效的方式来进行大规模交易，避免了高昂的手续费和潜在的市场操纵。

**2.1.1 原理**
TWAMM 有一个内置的 AMM，这个 AMM 和其他的 AMM 并没有什么不同，用户可以通过这个 AMM 直接进行现货交易，也可以向其中添加流动性。但是 TWAMM 同时还有两个 TWAP order pool，分别用来执行两个方向的 TWAP order，用户提交 order 时，指定交易的 token input 数量和时长，TWAMM 会将相同交易方向的 order 放入对应的 pool 中，并按照指定的交易速度自动进行交易。当用户的 order 被完全执行后，用户就可以拿出交易得到的 token。当然，在用户的 order 在被执行完成之前，用户也可以提前取消 order 或者修改 order 需要交易的 token 数量。
在以太坊中，智能合约只能由 EOA 地址主动发起交易触发执行，而不能自动执行。因此 TWAMM 需要由 EOA 账户定期发送交易来结算其 order pool 中待交易的 token，这样就需要一个 keeper 账号来执行这些交易。
当然，也可以让 TWAMM 在每次有用户与其交互时，自动结算 order pool，这样就省去了 keeper 的开销，这也是 DeFi 协议处理流式数据常用的方式。

**2.1.2 为什么说这种交易模式很难被三明治攻击？**
这种攻击很难实施，由于一个区块内 timestamp 不会改变，攻击者必须要在一个区块的最后一个交易中，将 pool 的价格拉高，这样下一个区块中的 TWAMM 结算才会受到影响。这就要求三明治攻击发生在多个区块中，这无疑会给攻击者带来很大的风险，因为其他套利者有可能在中间介入，导致攻击者遭受损失。
同时，因为套利者的存在，这样的价格操纵注定无法持久，由于 TWAP order 的特性它并不会在短时间内交易太多的 token，因此大部分情况损失也一定是有限的。

**2.1.3 V4 中的 TWAMM 工作流程**
（1）此 Hook 维护两个 TWAP order pool，分别表示两个交易方向的 TWAP order
（2）用户可以通过此 Hook 提交 TWAP order，需要指定交易的 token，数量以及时间长度
（3）此 Hook 注册 beforeSwap 和 beforeModifyPosition，每次用户交易或者调整仓位时，都会触发此 Hook
（4）被触发后，Hook 负责对 2 个 TWAP order pool 进行结算
（5）用户也可以在任意时刻手动触发结算
（6）用户可以取消或者修改 TWAP order 中的 token 数量

**2.1.4 实例详解**
TWAMM 注册三个阶段来进行 hooks 的逻辑调用，在 pool 初始化之前对 TWAMM 进行初始化，并在每次用户交易或者调整仓位时触发此 hook。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572134-7fa035b0-5c0a-4ed9-8b64-815e08458cef.jpeg#averageHue=%23292221&clientId=u4de60ac9-9e49-4&from=paste&id=uaa11f5a2&originHeight=683&originWidth=1095&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u178ec72d-a43d-47b4-8e20-56e2abf185f&title=)

用户可手动调用 TWAMM 中的 submitOrder 函数来提交自己需要执行的 order 到合约中。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572255-1ed1941d-87ce-40db-944a-68c4ffc1cc2b.jpeg#averageHue=%2339382f&clientId=u4de60ac9-9e49-4&from=paste&id=u99132e7b&originHeight=550&originWidth=1076&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=uf779e0f7-f0c5-4d59-bbdd-03b1482e500&title=)

在用户将自己需要执行的订单添加进合约后，在 pool 每次进行 swap 和 modifyPosition 操作时，都会自动进行 order 的执行。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572618-836b3a35-b95a-4c3a-a74a-64154960ca3e.jpeg#averageHue=%233e3c32&clientId=u4de60ac9-9e49-4&from=paste&id=u0d2e0d91&originHeight=368&originWidth=1144&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u359a3993-e066-4652-b437-2530458a946&title=)

每有用户调用 v4 的 swap 函数进行交易或 modifyPosition 函数更改仓位，都会触发 TWAMM 中的执行函数，函数中会调用内部函数\_executeTWAMMOrders 继续进行此前未完成 order 的执行。
\_executeTWAMMOrders 函数

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572577-80e9a8ff-07d2-4240-bbc3-1efe69e10c21.jpeg#averageHue=%2339362d&clientId=u4de60ac9-9e49-4&from=paste&id=u12981bd8&originHeight=402&originWidth=1128&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=uc3b7c508-e7e8-4369-b584-1c07d5615e5&title=)

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572638-a3d64f91-9378-416d-a687-c6be3235d37c.jpeg#averageHue=%233a3831&clientId=u4de60ac9-9e49-4&from=paste&id=ucaaed394&originHeight=824&originWidth=889&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u56503390-bed1-4156-8c99-bd127ecce57&title=)

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572733-57c82119-ef3f-4698-a258-d22f16d0a74c.jpeg#averageHue=%233c382f&clientId=u4de60ac9-9e49-4&from=paste&id=u4f0ad79f&originHeight=479&originWidth=842&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u9a3d4a4d-402a-4a3b-9edc-ec8d9898492&title=)

以上为更新订单的执行流程，在执行结束之后，会更新当前 twamm 的订单执行时间。

### 2.2 Limit Order

与立即按照最后的市场价格执行的市价单不同，限价单在达到预定价格后立即执行。基于自动做市商（AMM）的 DEX 大多默认选择市价单系统。对新人来说简单易懂。市价单要么被执行，要么因参数（如最大价格影响）而失败。而在限价单中，只有当资产价格达到限价时，订单才会成交，否则订单将保持未平仓状态。
例如，假设 ETH 目前在 ETH/DAI 池中的交易价格为 1 ETH = 1500 DAI。用户可以下一个止盈订单，主要内容是「如果 1 ETH = 2000 DAI，卖出我所有的 ETH」。如果达到这个价格，用户的 ETH 将自动以去中心化的方式完全在链上交换为 DAI。
在 Uniswap 以前的版本中，限价订单实际上是不可能的。大多数 AMM 只允许市场买入和卖出。而在 V4 版本中，由于 hook 的强大特性和可扩展性，让限价单在 v4 中存在了实现的基础。

**2.2.1 原理**
limit order 的设计原理相比于 twamm 来说，更加简单一些，目前实现了有关添加流动性的限价单，交易的限价单实现起来也会比较容易。
由于 v4 中存在 tickLower 和 tickUpper，并根据池子的交易情况变化，lower 和 upper 会发生改变，用户在进行添加流动性时并不想在当前价格进行添加，这时就可以利用 limit Order hook 来执行此需求，在 hook 中设定对应的价格，在每次 swap 结束后，hook 会对当前 pool 的价格做判断，若达到了设定的价格，则将对应的流动性进行添加获取收益。
**2.2.2 V4 中的 Limit Order 工作流程**

1. limit 维护着多个 epoch，作为不同 lower 和交易方向的限价单。
2. 用户通过这个 hook，提交自己的价格 lower 和交易方向，将限价单加入合约。
3. 此 Hook 注册 afterSwap，只在每次 swap 结束后价格发生变动时触发
4. 被触发后，Hook 校验当前价格区间，并从 epoch 中查验当前价格是否存在需要进行添加流动性的限价单。
5. 用户可以在任何时刻提款或直接添加流动性
   **2.2.3 实例详解**
   合约注册两个阶段触发 hook，在 Pool 初始化后对 hook 初始化，并在每次兑换结束后触发 hook 逻辑。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021572799-27e1541a-0ce7-409e-91cf-ddee7a684b3a.jpeg#averageHue=%2339332b&clientId=u4de60ac9-9e49-4&from=paste&id=u61ae9b79&originHeight=796&originWidth=1140&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u74e2529b-ea8d-4a00-882e-7bdb1904399&title=)

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573035-99491d49-b9aa-454c-a0fa-2ec175ad100b.jpeg#averageHue=%23282221&clientId=u4de60ac9-9e49-4&from=paste&id=u8a239dbd&originHeight=837&originWidth=1321&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u43ea0b6c-259d-480d-9d13-35f353f1a82&title=)

用户调用 place 函数传入想要添加流动性的数量，价格以及交易方向，hook 会先将用户想要添加的流动性加入 pool 中保存，随后为用户创建对应的限价单。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573047-7719f632-a7d2-4425-be42-c3d60d3cdff1.jpeg#averageHue=%2336332b&clientId=u4de60ac9-9e49-4&from=paste&id=u08dfaadd&originHeight=807&originWidth=1012&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u829b3cf4-561e-4a8b-be56-0da8a0bb03e&title=)

在每次 swap 结束后都会触发此 hook，根据当前价格区间将区间价格内存在的限价单完成添加流动性的操作。
![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573104-74e2920a-ca65-4bf6-8982-5fbfdbc52869.jpeg#averageHue=%233e362d&clientId=u4de60ac9-9e49-4&from=paste&id=ued652ed5&originHeight=820&originWidth=1043&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=ub80598d2-2562-4d43-952b-89ca2221c0b&title=)

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573159-383d46fe-34f6-4074-b835-0794e6f56d18.jpeg#averageHue=%233f342b&clientId=u4de60ac9-9e49-4&from=paste&id=u4bbc0864&originHeight=914&originWidth=1194&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u393422b3-bcf0-45d0-9a41-1d5a71a7717&title=)

用户可在限价单完成之前调用 kill 函数取消限价单并获得此次添加流动性获得的收益。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573274-fdbe9420-4492-4ae4-a513-38fbdd5067be.jpeg#averageHue=%23403a31&clientId=u4de60ac9-9e49-4&from=paste&id=u75ca0506&originHeight=631&originWidth=1226&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u43706389-6d59-4beb-8e16-6239b2712d7&title=)

用户可在想要移除流动性时调用 withdraw 函数，即可提取想要的流动性。
总的来说，此 limit order hook 为用户提供了一个更便捷的途径，用户可设定自己想要添加流动性时的价格，在池子中价格区间到达该价格后，hook 自动为用户到 pool 中执行添加流动性的操作，且用户可以随时取消和提取流动性。

除 TWAMM 和 Limit Order 外，基于 Hook 还可以实现 LP 复投、动态手续费变化等功能，因为篇幅原因，我们将在后续的分析中展开介绍：

- LP 复投：LP 用户可利用 hook 来进行流动性的添加，修改和移除，hook 可注册 afterSwap 和 afterModifyPosition 来进行调用。由于用户统一使用 hook 来进行流动性的添加，所以在 poolManager 中添加流动性的地址只有 hook, hook 可在每次触发时查验当前时间，每经过一定时间段后，选择提取流动性奖励并将获得的 lp 代币再次到 pool 中添加流动性，从而自动化优化用户收益。
- 动态手续费变化：hook 可注册 beforeSwap 等接口，来进行交换之前的动态手续费修改。动态手续费可以不是简单地根据时间线性变化，可以根据单笔 swap 产生的 tick 跳跃数量来量化波动率，从而动态改变手续费，实现对于 LP 无常风险的对冲。从而减少因为交易产生的无常损失对流动性提供者造成的影响。

![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573515-e7ff12b2-3bb7-45b5-90b4-55f9caf3acc5.jpeg#averageHue=%23282221&clientId=u4de60ac9-9e49-4&from=paste&id=u7efae52d&originHeight=558&originWidth=1001&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u27ebb5a1-16af-474a-a9a6-a368d121c1d&title=)

## 三、Hooks 安全最佳实践

**减少 require 和 revert 的使用**
在有关 pool 对 hooks 调用的函数逻辑中，尽量减少回退语句的使用，由于 pool 合约与 hooks 合约存在共通关系，在 hooks 中发生交易回滚后，会导致 pool 中的交易同样回退，如果在 hooks 中出现与 pool 中正常交易不相关的回退语句，可能会导致用户无法正常使用 pool 中的功能。
![](https://cdn.nlark.com/yuque/0/2023/jpeg/97322/1693021573668-df3a61be-8089-4293-9f37-df5c232685a2.jpeg#averageHue=%23282322&clientId=u4de60ac9-9e49-4&from=paste&id=u2c9d3229&originHeight=140&originWidth=884&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u0637afae-d624-4c60-9998-096533e342a&title=)
**避免使用自毁函数**
避免在 hook 中使用 selfdestruct 函数，如果 hook 中调用到了自毁函数，不仅会导致 hook 中的逻辑出现问题，并且会导致 pool 中的功能也无法正常进行，使整个 pool 池中的资产丢失并且功能无法正常进行。
**严格进行权限控制**
严格控制 hook 合约中的权限，避免出现权限过大的角色，并对特权角色进行多签管理，防止出现单点攻击。避免存在特权角色可任意修改合约状态变量的情况，可能会出现逻辑错误导致整个交易发生回滚，影响 pool 的正常使用。验证最小权限原则，我们应当使用 openzeppelin 的 AccessControl 合约来实现控制访问更细粒度的权限，因为该实践限制每个系统组件遵循最小权限原则。
**做好防重入限制**
hooks 作为 Pool 的外部扩展代码，同样应该注意合约中可能出现的重入攻击，例如限价单中在进行原生代币转账时可能出现的重入，导致合约资产出现损失。在调用外部合约或所谓的「检查 - 生效 - 交互」模式之前检查并尝试更新所有状态。这样，即使重入，也不会产生任何影响，因为所有状态都已完成更新。
**合约升级控制**
有些开发者可能会使用代理合约以便后续对 hook 的逻辑进行变动和升级，但也因此需要注意到合约升级方面可能存在的问题。首先，hook 中即使采取代理模式，代理合约中也需要声明对应的阶段，例如 beforeSwap 等，否则 pool 无法校验到正确的返回值。其次，在调用 delegatecall 之前检查目标合约是否存在，Solidity 不会替我们执行此检查，忽略检查可能会导致意外行为和安全问题，仔细考虑变量声明的顺序，因为会出现变量打包存储同一个插槽、影响 gas 成本、内存布局、delegate 调用结果等问题。
