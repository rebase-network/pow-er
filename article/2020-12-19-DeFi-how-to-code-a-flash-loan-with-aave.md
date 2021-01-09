---
category: DeFi
title: How To Code a flash load with AAVE
date: "2020-12-19 00:00:00"
---

# 开发者如何玩转 Aave 闪电贷

原文链接：https://finematics.com/how-to-code-a-flash-loan-with-aave/

作者：jakub

如何通过简单的几步就能完成闪电贷的编码呢？又如何与 Aave 的智能合约进行交互呢？你将在本教程中找到这些问题的答案。

## 什么是闪电贷?

闪电贷是一种功能，它允许你从指定的智能合约池中借入任何可用的资产，而无需抵押品。闪电贷款是 DeFi 中有用的基础，因为它们可用于套利、交换抵押品和自我清算等。

闪电贷虽然最初是由 [Marble protocol](https://github.com/marbleprotocol/flash-lending) 引入的，但由 Aave 和 Dydx 普及。

闪电贷必须在同一区块链交易中进行借贷和偿还。

如果你想了解更多关于闪电贷的信息，请在这里查看我们[关于闪电贷的文章](https://finematics.com/flash-loans-explained/)。

## 开始前

本指南要求对 Solidity 智能合约和如何使用 Remix 有基本的了解。你可以找到[Remix 文档](https://remix-ide.readthedocs.io/en/latest/)。

还需要安装 Metamask 钱包并切换到 Kovan 测试网。

![img]

要在 Ethereum 上执行智能合约，你的钱包里还得有一些 ETH。你可以通过访问https://faucet.kovan.network/ 并使用你的 Github 账户登录来获得一些 Kovan testnet ETH。你每天可以申请 1 次 testnet ETH。

我们还需要一些 DAI。要获得 DAI，请访问[aave faucet](http://testnet.aave.com/faucet)并点击 DAI，然后点击 “提交” 按钮。

好了，最后一步就是设置你的 Remix 环境。前往 http://remix.ethereum.org，选择0.6.6+版本的Solidity编译器。

![img]

## 闪电贷智能合约

先让我们从创建一个基本的智能合约文件开始。

![img]

这是我们需要的第一段代码。

```
pragma solidity ^0.6.6;

import "https://github.com/aave/flashloan-box/blob/Remix/contracts/aave/FlashLoanReceiverBase.sol";
import "https://github.com/aave/flashloan-box/blob/Remix/contracts/aave/ILendingPoolAddressesProvider.sol";
import "https://github.com/aave/flashloan-box/blob/Remix/contracts/aave/ILendingPool.sol";

contract Flashloan is FlashLoanReceiverBase {
    constructor(address _addressProvider) FlashLoanReceiverBase(_addressProvider) public {}
}
```

这将导入所有必要的依赖关系，并创建一个 `FlashLoan.sol` 智能合约，该合约继承自`FlashLoanReceiverBase`，它是一个抽象合约，提供了一些有用的东西，如偿还闪电贷的方式。

`Flashloan.sol`构造函数接受 Aave 的一个贷款池提供者的地址。我们会稍后将介绍。

现在你可以点击编译按钮了。

![img]

remix 会下载所有相关的代码，并会出现一个错误，因为我们还没有从`FlashLoanReceiverBase`中实现所有必要的方法。

这 2 个缺失的函数了，第一个是我们的 `flashLoan` 函数，它将被用来触发 闪电贷。另一个是 `FlashLoanReceiverBase`中缺少的方法--`executeOperation`，它将在触发闪电贷方法后被调用。

## flashloan 方法

我们先来添加 `flashLoan` 函数。

下面是代码片段。

```
function flashloan(address _asset) public onlyOwner {
    bytes memory data = "";
    uint amount = 1000000000000000000;

    ILendingPool lendingPool = ILendingPool(addressesProvider.getLendingPool());
    lendingPool.flashLoan(address(this), _asset, amount, data);
}
```

flashLoan 的参数`_asset`是我们要用闪电贷借款的资产地址，比如 ETH 或 DAI。

我们将函数定义为`onlyOwner`，所以只有合约的所有者才能调用该函数。

`uint amount = 1000000000000000000;`

这里，我们定义的借款金额是最小的单位--wei 即 10^18，所以这个值等于 1，如果把 DAI 地址传给`_asset`，我们就会借到 1 个 DAI，如果把 ETH 地址传过去，我们就会借到 1 个 ETH。

现在，我们可以使用 Aave 提供的 `ILendingPool`接口，调用`flashLoan`函数，其中包含所有需要的参数，如我们想要借入的资产、该资产的金额和一个额外的`data`参数。

即使添加了 `flashLoan`函数，代码仍然无法编译，因为我们必须添加`executeOperation`函数。

## executeOperation 方法

`executeOperation`函数将被`LendingPool` 合约在闪电贷中请求有效的资产后被调用。

这里是 `executeOperation` 函数的代码片段。

```
function executeOperation(
        address _reserve,
        uint256 _amount,
        uint256 _fee,
        bytes calldata _params
    )
        external
        override
    {
        require(_amount &lt;= getBalanceInternal(address(this), _reserve), "Invalid balance, was the flashLoan successful?");

        // Your logic goes here.

        uint totalDebt = _amount.add(_fee);
        transferFundsBackToPoolInternal(_reserve, totalDebt);
    }
```

在使用 `flashLoan` 函数触发有效的闪电贷后，`executeOperation` 函数的所有参数都将被自动传递。

`require` 是用确保收到的闪电贷金额是否正确。

接下来，我们可以插入任何想要执行的逻辑。在这个步骤中，我们拥有了来自闪电贷的所有可用资金，因此，我们可以尝试套利机会。

我们在用完闪电贷后，就需要偿还资金了。

`uint totalDebt = _amount.add(_fee);`

在这里，我们要计算还多少钱，也就是 **借款金额 + 借款金额的 0.09%**。 **Aave 的闪电贷需要手续费**。

最后一步就是调用`transferFundsBackToPoolInternal`来偿还闪电贷。

现在，我们可以尝试再次编译代码。这一次，一切都应该编译好了，是时候触发`flashLoan`函数了。

## 运行代码

我们的闪电贷智能合约已经写好了，可以部署了。

我们先来准备 2 个必要的东西。

- `LendingPoolAddressesProvider` —— 为了部署合约，我们需要找到 Aave 在 Kovan 测试网上的借贷合约的地址。它地址是`0x506B0B2CF20FAA8f38a4E2B524EE43e1f4458Cc5`。你可以在[aave 文档](https://docs.aave.com/developers/deployed-contracts) 找到所有的地址。

- DAI 地址，我们需要DAI（或你想借用的任何其他资产）在 Kovan 测试网的合约地址。它的地址是`0xFf795577d9AC8bD7D90Ee22b6C1703490b6512FD`。

是时候部署智能合约了。

1.  转到 Remix 上的 “Deploy & Run Transactions”。

2.  选择 “Injected Web3” (确保你的 Metamask 切换到 Kovan)

3.  选择要部署的合约 `FlashLoan.sol`。

4.  将前面几步找到的 `LendingPoolAddressesProvider` 传给 “Deploy” 按钮旁边的字段。

5.  点击 “Deploy” 按钮

6.  通过 Metamask 确认你的 Ethereum 交易

![img]

如果一切顺利，太好了! 现在我们已经在 Ethereum 测试网 Kovan 上部署了 `FlashLoan.sol`智能合约。

下面是运行智能合约的关键步骤。我们必须向刚刚创建的智能合约发送一些 DAI，以便能够支付闪电贷费用。

你可以通过检查你之前批准的 Metamask 交易来找到你的智能合约地址。创建智能合约的交易应该类似于这个[交易](https://kovan.etherscan.io/tx/0x2da8a86a728e502c30dead492dfd0c468f03c851e5843e3bf8b5cee1fb52dbde)。

从这个交易中，你可以找到智能合约地址（这个例子中地址是`0x27016b23BEE0553A4aAa89b25Be58b93Fe647BBe`）。

是时候从部署的合约中触发的`flashLoan`函数了。记住要传递一个正确的资产地址。在这个的例子中，它是 Kovan testnet 上的 DAI 地址。

![img]

在触发 flashLoan 函数并通过 Metamask 接受 Ethereum 交易后，你应该会看到一个类似于这个[交易](https://kovan.etherscan.io/tx/0xd2dfb7f42dde9e0af730e1afd9ff38177b3eef38448ddc0093cdf62fb234c6c7)。

恭喜你！你刚完成了一笔闪电贷交易!

如果你喜欢本教程，你还可以在[Youtube](https://www.youtube.com/c/finematics)和[Twitter](https://twitter.com/finematics)上关注 Finematics。
