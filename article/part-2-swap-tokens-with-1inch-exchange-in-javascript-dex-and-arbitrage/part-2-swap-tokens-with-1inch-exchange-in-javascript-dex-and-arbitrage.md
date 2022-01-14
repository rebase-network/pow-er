原文：https://ethereumdev.io/swap-tokens-with-1inch-exchange-in-javascript-dex-and-arbitrage-part-2/

# Swap tokens with 1inch Exchange in JavaScript: DEX and Arbitrage part 2

对于本教程，将了解如何使用 1inch DEX 聚合器执行交易使用 web3.js 库的 JavaScript 中的。通过本教程，你将了解如何在以太坊区块链上直接交换 ERC20 token和以太币。

本文是第 2 部分，在上一篇向你介绍了如何获得交易报价：获取你要出售的token所获得的token数量。本文，我们来看一看如何用 JavaScript 执行交易。

[img]

要完成 1inch 的 DEX 聚合器上的兑换，我们需要做3件事：

- 获取当前汇率如第 1 部分所示

- 授权(Approve)我们要兑换的 ERC20 token的数量，以便 1inch 智能合约可以访问你的资金。

- 通过使用在步骤 1 中获得的参数调用 swap 方法进行交易。

但是在开始之前，我们需要定义一些变量和一些辅助函数，使代码更简单。

## Setup

在本教程中，我们将使用 [ganache-cli 分叉(fork)当前的区块链状态](https://ethereumdev.io/testing-your-smart-contract-with-existing-protocols-ganache-fork/)，并提前在1个地址上充值了很多 DAI。在我们的示例中，地址是 `0x78bc49be7bae5e0eec08780c86f0e8278b8b035b`。我们还将 gas 限制设置为非常高，因此在测试过程中不至于出现 out-of-gas 问题，也不需要在每次交易前估算 gas。启动分叉区块链的命令行是：

```
ganache-cli -f https://mainnet.infura.io/v3/[YOUR INFURA KEY] -d -i 66 --unlock 0x78bc49be7bae5e0eec08780c86f0e8278b8b035b -l 8000000
```

## 实战 - 执行多DEX兑换方案

在本教程中，我们使用1inch聚合器将1000 DAI兑换为ETH。首先定义一些变量，例如合约地址、ABI等。

```js
var Web3 = require('web3');
const BigNumber = require('bignumber.js');

const oneSplitABI = require('./abis/onesplit.json');
const onesplitAddress = "0xC586BeF4a0992C495Cf22e1aeEE4E446CECDee0E"; // 1plit contract address on Main net

const erc20ABI = require('./abis/erc20.json');
const daiAddress = "0x6b175474e89094c44da98b954eedeac495271d0f"; // DAI ERC20 contract address on Main net

const fromAddress = "0x4d10ae710Bd8D1C31bd7465c8CBC3add6F279E81";

const fromToken = daiAddress;
const fromTokenDecimals = 18;

const toToken = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'; // ETH
const toTokenDecimals = 18;

const amountToExchange = new BigNumber(1000);

const web3 = new Web3(new Web3.providers.HttpProvider('http://127.0.0.1:8545'));

const onesplitContract = new web3.eth.Contract(oneSplitABI, onesplitAddress);
const daiToken = new web3.eth.Contract(erc20ABI, fromToken);
```

同时写一个辅助函数来等待交易确认：

```js
function sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function waitTransaction(txHash) {
    let tx = null;
    while (tx == null) {
        tx = await web3.eth.getTransactionReceipt(txHash);
        await sleep(2000);
    }
    console.log("Transaction " + txHash + " was mined.");
    return (tx.status);
}
```

## 获取汇率

我们在之前已经获得了交易汇率，现在把代码变的更可读。

函数`getQuote` 返回一个包含所有参数的对象，以使用[第一部分中详细介绍的兑换函数]()。

```js
async function getQuote(fromToken, toToken, amount, callback) {
    let quote = null;
    try {
        quote = await onesplitContract.methods.getExpectedReturn(fromToken, toToken, amount, 100, 0).call();
    } catch (error) {
        console.log('Impossible to get the quote', error)
    }
    console.log("Trade From: " + fromToken)
    console.log("Trade To: " + toToken);
    console.log("Trade Amount: " + amountToExchange);
    console.log(new BigNumber(quote.returnAmount).shiftedBy(-fromTokenDecimals).toString());
    console.log("Using Dexes:");
    for (let index = 0; index < quote.distribution.length; index++) {
        console.log(oneSplitDexes[index] + ": " + quote.distribution[index] + "%");
    }
    callback(quote);
}
```

## Approve the spending of the token

一旦获得了第1部分中所述的token交换率，我们首先需要批准1inch dex聚合器合约来花费我们的token。如你所知，ERC20 token标准不允许将token发送到智能合约并在一次交易中触发它的操作。我们编写了一个简单的函数，该函数调用ERC20合约实例批准函数，并等待使用进行交易。正好用到辅助函数 waitTransaction 。

一旦我们得到了兑换token的比率，接下来需要授权1inch可以操作我们持有的token，ERC20 token标准不允许在一次交易中向合约发送token并触发它的一个动作。我们写了一个简单的函数，调用`approval`函数，并使用 waitTransaction 等待交易确认。

```js
function approveToken(tokenInstance, receiver, amount, callback) {
    tokenInstance.methods.approve(receiver, amount).send({ from: fromAddress }, async function(error, txHash) {
        if (error) {
            console.log("ERC20 could not be approved", error);
            return;
        }
        console.log("ERC20 token approved to " + receiver);
        const status = await waitTransaction(txHash);
        if (!status) {
            console.log("Approval transaction failed.");
            return;
        }
        callback();
    })
}
```

注意，授权额度可以远远高于当前实际需要的数量，这样后续就不再需要执行这个环节的操作了。

## Make the swap

接下来就可以调用1inch聚合器的swap函数了。在下面的代码中，我们在调用`swap`函数执行交易后，等待交易确认，并在交易确认后，显示转出账户的eth余额和dai余额。

```js
let amountWithDecimals = new BigNumber(amountToExchange).shiftedBy(fromTokenDecimals).toFixed()

getQuote(fromToken, toToken, amountWithDecimals, function(quote) {
    approveToken(daiToken, onesplitAddress, amountWithDecimals, async function() {
        // We get the balance before the swap just for logging purpose
        let ethBalanceBefore = await web3.eth.getBalance(fromAddress);
        let daiBalanceBefore = await daiToken.methods.balanceOf(fromAddress).call();
        onesplitContract.methods.swap(fromToken, toToken, amountWithDecimals, quote.returnAmount, quote.distribution, 0).send({ from: fromAddress, gas: 8000000 }, async function(error, txHash) {
            if (error) {
                console.log("Could not complete the swap", error);
                return;
            }
            const status = await waitTransaction(txHash);
            // We check the final balances after the swap for logging purpose
            let ethBalanceAfter = await web3.eth.getBalance(fromAddress);
            let daiBalanceAfter = await daiToken.methods.balanceOf(fromAddress).call();
            console.log("Final balances:")
            console.log("Change in ETH balance", new BigNumber(ethBalanceAfter).minus(ethBalanceBefore).shiftedBy(-fromTokenDecimals).toFixed(2));
            console.log("Change in DAI balance", new BigNumber(daiBalanceAfter).minus(daiBalanceBefore).shiftedBy(-fromTokenDecimals).toFixed(2));
        });
    });
});
```

最后的执行结果看起来是下面这样的：

[img]

正如你看到的，我们用1000 DAI换回来5.85 ETH。 (现在请开始哭)

代码已在 github 开源，尝试下面的操作来执行操作：

```
git clone https://github.com/jdourlens/ethereumdevio-dex-tutorial && cd ethereumdevio-dex-tutorial/part2

npm install

node index.js
```

在这个过程中，你可能会遇到的这样一个错误提示：“**VM Exception while processing transaction: revert OneSplit: actual return amount is less than minReturn**”。这表示链上的报价已经更新。如果想避免这种情况发生，你可以在代码中引入一个滑点，根据交易金额，将minReturn参数减小1%或3%。

## 1inch Dex聚合器使用小结

1inch提供了出色的链上DEX聚合实现，可以在一个交易内利用多个DEX实现最优的兑换策略。1inch的API使用也很简单，只需要用getExpectedReturn估算兑换方案， 然后使用swap执行兑换方案，就可以得到最好的兑换结果。 您不必总是用Ether交换，可以一起交换2个ERC20代币，甚至可以用Wrapped Ether交换。

http://blog.hubwiz.com/2020/11/29/1inch-arbitrage/
