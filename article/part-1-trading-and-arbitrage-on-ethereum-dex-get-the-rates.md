原文：https://ethereumdev.io/trading-and-arbitrage-on-ethereum-dex-get-the-rates-part-1/

# 在 DEX 进行交易和套利：获取汇率 (part 1)

## 在 1inch 上获取最优兑换

在本系列教程中，我们探讨如何围绕交易构建解决方案，并使用 Ethereum 去中心化交易所（DEX）制作简单的套利机器人。
我们会用到 Javascript、Solidity 和 1inch dex 聚合器和闪电贷。这个教程可能不会让你发财，但会让你获得利用 Ethereum DeFi 生态系统构建强大应用的钥匙。如果你喜欢 Python，可以试试 https://wolfdefi.com/projects/1-inch-python-trades/。

我们将该系列分为 2 部分：

- 获取链上 token 兑换汇率

- [使用 JavaScript 和 1inch dex 聚合器进行兑换](https://ethereumdev.io/swap-tokens-with-1inch-exchange-in-javascript-dex-and-arbitrage-part-2/)

[img]

## 什么是去中心化交易所聚合器?

DEX 聚合器是一个平台，它将搜索一组 DEX，以寻找在给定时间和数量下执行交易的最佳价格。

## 什么是套利?

套利基本上是在一个市场上买入某样东西，同时在另一个市场上以更高的价格卖出，从而从暂时的价格差异中获利。

在本教程中，我们将从一个 DEX 买入 token，并在另一个 DEX 上以更高的价格卖出。在区块链上，最主要也是最古老的套利机会是通过跨去中心化和中心化交易所的交易。

## 1inch DEX 聚合器

1inch 交易所是由[Anton Bukov](https://github.com/k06a)和 [Sergej Kunz](https://github.com/deacix)开发的去中心化交易所聚合器，通过一次交易将订单在多个 DEX 之间拆分，给用户提供最好的汇率。
1inch 的智能合约是开源的，可以[在 Github 上查看](https://github.com/1inch-exchange/1inchProtocol)。我们将看到如何使用智能合约发现交易机会。 你可以在 https://1inch.exchange/ 访问 1inch 用户界面。
[img]

在 1inch 执行交易，过程其实很简单：

- 根据输入的 token 或 ETH 数量，获得预期可兑换的 token 数量
- 授权（Approve）交易所使用你的 token
- 使用第一步的获取的 token 数量进行交易

我们首先仔细了解一下 1inch 的智能合约，让我们感兴趣的是这两个功能：

- `getExpectedReturn()`
- `swap()`

## 获取预期可兑换的 token

`getExpectedReturn` 函数不会改变链的状态，只要你有连接到区块链的节点，就可以随意调用，不需要任何费用。

它接受交易参数，并返回将获得的预期的 token 数量，以及交易如何在 dex 之间分布。

```js
function getExpectedReturn(
    IERC20 fromToken,
    IERC20 toToken,
    uint256 amount,
    uint256 parts,
    uint256 disableFlags
) public view
returns(
    uint256 returnAmount,
    uint256[] memory distribution
);
```

这个 方法接收 5 个参数：

- fromToken： 当前拥有的 token 的地址
- toToken： 要交换的 token 的地址
- amount： 想要交换的 token 数量
- parts： 卖出数量拆分成多少份进行最优分布的估算。查看`distribution` 可以了解更多细节，默认是 100
- disableFlags：标记位，用于调整 1inch 的算法，例如可设置禁用某个特定的 DEX

这个方法有 2 个返回值：

- returnAmount：执行交易后将收到的 token 数量。

- distribution：一个 uint256 类型的数组，代表交易在不同 DEX 中的分布情况。 例如，parts 设置为 100，成交额度的 25％在 Kyber 的，成交额度的 75％在 Uniswap，那么 `distribution` 可能看起来是这样：[75, 25, 0, 0, …]。

目前 1inch 支持的交易所和排序（与 distribution 对应）如下：

```
[
    "Uniswap","Kyber", "Bancor","Oasis",
    "CurveCompound","CurveUsdt","CurveY",
    "Binance","Synthetix",
    "UniswapCompound","UniswapChai","UniswapAave"
]
```

注意：如果你想交易 Eth 而不是 ERC20 token，fromToken 需要设置为特殊的值 `0x0`或
`0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`。

`getExpectedReturn`函数的返回值非常重要，因为接下来需要利用它来执行实际的链上兑换操作。

## 执行多 DEX 兑换交易

要执行链上 token 兑换交易，就需要使用 1inch 合约提供的另一个函数 swap。调用 swap 时，需要传入我们之前从`getExpectedReturn`返回的数据，并且承担必要的 gas 开销。如果要卖出的是 ERC20token，那么还需要先授权 1inch 合约 可以操作你持有的待卖出 token。`swap`函数的定义如下：

```js
function swap(
    IERC20 fromToken,
    IERC20 toToken,
    uint256 amount,
    uint256 minReturn,
    uint256[] memory distribution,
    uint256 disableFlags
 ) public payable;
```

swap 函数接收 6 个参数：

- fromToken：待卖出 token 的地址
- toToken：待买入 token 的地址
- amount：待卖出 token 的数量
- minReturn：期望得到的待买入 token 的最少数量
- distribution：兑换交易拆分分布数组
- parts：执行估算时的拆分数量，默认值是 100
- disableFlags：标记位，例如可设置禁用某个特定的 DEX

## 进行第一笔兑换交易

We’ll now try to use what we saw to get the rate for our first automated trade using JavaScript and a smart contract. If you did not [read our guide about setting up web3js you should read it](https://ethereumdev.io/setup-web3js-to-use-the-ethereum-blockchain-in-javascript/).

To make it easy for you, we published the [source code on Github](https://github.com/jdourlens/ethereumdevio-dex-tutorial) so you can easily get the code on your side:

分析完了 1inch 的关键方法，我们将进行第一笔兑换交易，代码已在 github 开源，尝试下面的操作来执行操作：

```
git clone https://github.com/jdourlens/ethereumdevio-dex-tutorial && cd ethereumdevio-dex-tutorial/part1

npm install

node index.js
```

`index.js` 代码如下：

```
var Web3 = require('web3');
const BigNumber = require('bignumber.js');

const oneSplitABI = require('./abis/onesplit.json');
const onesplitAddress = "0xC586BeF4a0992C495Cf22e1aeEE4E446CECDee0E"; // 1plit contract address on Main net

const fromToken = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'; // ETHEREUM
const fromTokenDecimals = 18;

const toToken = '0x6b175474e89094c44da98b954eedeac495271d0f'; // DAI Token
const toTokenDecimals = 18;

const amountToExchange = 1

const web3 = new Web3('http://127.0.0.1:8545');

const onesplitContract = new web3.eth.Contract(oneSplitABI, onesplitAddress);

const oneSplitDexes = [
    "Uniswap",
    "Kyber",
    "Bancor",
    "Oasis",
    "CurveCompound",
    "CurveUsdt",
    "CurveY",
    "Binance",
    "Synthetix",
    "UniswapCompound",
    "UniswapChai",
    "UniswapAave"
]


onesplitContract.methods.getExpectedReturn(fromToken, toToken, new BigNumber(amountToExchange).shiftedBy(fromTokenDecimals).toString(), 100, 0).call({ from: '0x9759A6Ac90977b93B58547b4A71c78317f391A28' }, function (error, result) {
    if (error) {
        console.log(error)
        return;
    }
    console.log("Trade From: " + fromToken)
    console.log("Trade To: " + toToken);
    console.log("Trade Amount: " + amountToExchange);
    console.log(new BigNumber(result.returnAmount).shiftedBy(-fromTokenDecimals).toString());
    console.log("Using Dexes:");
    for (let index = 0; index < result.distribution.length; index++) {
        console.log(oneSplitDexes[index] + ": " + result.distribution[index] + "%");
    }
});
```

第 4 行：加载 ABI 以便实例化 web3.js 和 1inch 合约实例

第 19 行：该数组指定要使用的 DEX

第 35 行：调用`getExpectedReturn`函数获取兑换方案

代码执行后返回结果类似下面这样：

[img]

此时卖出 1 个 eth，1inch 可以买到 148.47DAI，而 Coinbase 是 148.12。1inch 给出的最佳兑换方案是通过 Uniswap 兑换 96%，再通过 Bancor 兑换 4%，这样可以得到 148.47 DAI，这样比单独通过 Uniswap 或 Bancor 进行兑换都划算。

注意，这个价格不能提供给智能合约的 Oracle，由于各种各样的错误，DEX 可以提供非常低的价格，因此可能会严重操纵这个价格。

能够轻松使用 DEX 聚合器是 DeFi 的一项很棒的功能。 由于大多数协议都是开放的，因此你只需要了解它们即可使用 DeFi 的功能。
