1.  [Architecture](#architecture)
1.  [Contracts](#contracts)
    1.  [Forwarder](#forwarder)
1.  [Contract Interactions](#contract-interactions)
    1.  [Fee Abstraction](#fee-abstraction)
1.  [Events](#events)
    1.  [Forwarder events](#forwarder-events)
1.  [Miscellaneous](#miscellaneous)
    1.  [Optimizing calldata](#optimizing-calldata)

# Architecture

![Forwarder](https://github.com/0xProject/0x-protocol-specification/blob/921082aa00dcffc6ecad00b68c3984db1893dce6/v2/0x_v2_forwarder.png?raw=true)

The Forwarding contract acts as a middleman between the user and the 0x Exchange contract. Its purpose is to perform a number of useful actions on the users behalf. Conveniently reducing the number of steps and transactions.

0x Exchange (see Asset Proxies) works as an operator which is authorised to move tokens as per the logic defined in 0x Exchange. As Ether is not a token and cannot be moved natively by a third party operator, Wrapped Ether (WETH) is introduced to adapt Ether to work in this environment. Wrapped Ether requires a user to deposit (transaction) and approve (transaction) the Asset Proxy operator.

When performing an exchange on 0x, fees may be required to be paid by the maker and takers in ZRX token. The Asset Proxy needs approval to move these tokens (transaction) and ZRX tokens need to be pre-purchased (transaction).

A user is then able to find an order and perform a TokenA/WETH fillOrder on the 0x Exchange contract (transaction).

For first time purchasers of tokens, this is non-trivial setup that is unavoidable due to the design of the ERC20 standard. The Forwarding contracts purpose is to enable users to buy tokens in one transaction.

# Contracts

## Forwarder

The Forwarder contains all of the following logic:

-   Depositing and Withdrawing WETH
-   Performing the exchange via 0x
-   Performing ZRX fee abstraction
-   Transferring the Tokens to the user

There are two public entry points for users of the Forwarding contract, `marketBuyTokensWithEth` and `marketSellEthForERC20`.

The implementation of these methods follows a 4 step process:

1.  Calculate any fees required to fill orders
2.  If there are any fees, buy this amount of ZRX fees using feeOrders
3.  Fill the orders and transfer all tokens back to msg.sender
4.  If feeProportion and feeRecipient are supplied deduct the fees from the amount traded

### marketBuyTokensWithEth

This function buys a specific amount of assets (ERC20 or ERC721) given a set of TokenA/WETH orders. If fees are required on the orders, then feeOrders of ZRX/WETH are supplied and this is used to purchase the fees required. The user specifies the amount of tokens they wish to buy with the makerAssetAmount parameter. makerAssetAmount must equal the length of orders if buying ERC721 tokens.

```solidity
    /// @dev Buys the exact amount of assets (ERC20 and ERC721), performing fee abstraction if required.
    ///      All order assets must be of the same type. Deducts a proportional fee to fee recipient.
    ///      This function is payable and will convert all incoming ETH into WETH and perform the trade on behalf of the caller.
    ///      The caller is sent all assets from the fill of orders. This function will revert unless the requested amount of assets are purchased.
    ///      Any excess ETH sent will be returned to the caller
    /// @param orders An array of Order struct containing order specifications.
    /// @param signatures An array of Proof that order has been created by maker.
    /// @param feeOrders An array of Order struct containing order specifications for fees.
    /// @param makerTokenFillAmount The amount of maker asset to buy.
    /// @param feeSignatures An array of Proof that order has been created by maker for the fee orders.
    /// @param feeProportion A proportion deducted off the ETH spent and sent to feeRecipient. The maximum value for this
    ///        is 1000, aka 10%. Supports up to 2 decimal places. I.e 0.59% is 59.
    /// @param feeRecipient An address of the fee recipient whom receives feeProportion of ETH.
    /// @return FillResults amounts filled and fees paid by maker and taker.
    function marketBuyTokensWithEth(
        LibOrder.Order[] memory orders,
        bytes[] memory signatures,
        LibOrder.Order[] memory feeOrders,
        bytes[] memory feeSignatures,
        uint256 makerTokenFillAmount,
        uint16  feeProportion,
        address feeRecipient
    )
        payable
        public
        returns (FillResults memory totalFillResults)
```

As filling these orders is a moving target (a partial fill can occur while this transaction is pending) it is safe to send in additional orders and additional Ether. The makerAssetAmount is the total amount of tokens the user wishes to purchase, and any additional Ether is returned to the user. This allows for the possibility of safely sending in 100 orders and 100 Ether to purchase 1 token, with all unspent Ether being returned.

#### marketBuyTokensWithEth when ZRX is the asset

Buying an exact amount of ZRX assets is a special case as fees (paid in ZRX) need to be accounted for. If a user wishes to purchase 100 ZRX and there is a 2 ZRX fee on this order, the Forwarding contract guarantees they will receive the expected amount (100 ZRX). To accomplish this we calculate an adjusted exchange rate (taking into account the fees). There is no need to calculate and buy fee tokens before executing this asset exchange as fees can be purchased from the same pool of orders. In this scenario the caller can supply an empty fee orders and fee signatures array.

### marketSellEthForERC20

This function attempts to buy as many ERC20 tokens as possible given the amount of Ether sent in by performing a 0x marketSell. ERC721 tokens are not supported in this function. If fees are required on the orders, then feeOrders of ZRX/WETH are supplied and this is used to purchase the fees required.

```solidity
    /// @dev Market sells ETH for ERC20 tokens, performing fee abstraction if required. This does not support ERC721 tokens. This function is payable
    ///      and will convert all incoming ETH into WETH and perform the trade on behalf of the caller.
    ///      This function allows for a deduction of a proportion of incoming ETH sent to the feeRecipient.
    ///      The caller is sent all tokens from the operation.
    ///      If the purchased token amount does not meet an acceptable threshold then this function reverts.
    /// @param orders An array of Order struct containing order specifications.
    /// @param signatures An array of Proof that order has been created by maker.
    /// @param feeOrders An array of Order struct containing order specifications for fees.
    /// @param feeSignatures An array of Proof that order has been created by maker for the fee orders.
    /// @param feeProportion A proportion deducted off the incoming ETH and sent to feeRecipient. The maximum value for this
    ///        is 1000, aka 10%. Supports up to 2 decimal places. I.e 0.59% is 59.
    /// @param feeRecipient An address of the fee recipient whom receives feeProportion of ETH.
    /// @return FillResults amounts filled and fees paid by maker and taker.
    function marketSellEthForERC20(
        LibOrder.Order[] memory orders,
        bytes[] memory signatures,
        LibOrder.Order[] memory feeOrders,
        bytes[] memory feeSignatures,
        uint16  feeProportion,
        address feeRecipient
    )
        payable
        public
        returns (FillResults memory totalFillResults)
```

Fees as defined by feeProportion are taken out before executing the market buy, it is calculated based on the total amount of Ether sent in. This function will revert unless all Ether is spent.

#### marketSellEthForERC20 when ZRX is the asset

As is similiar with marketBuyTokensWithEth there is no need to calculate and buy ZRX fees as these can be deducted from the same pool of order tokens. Thus we have a special case when the token being bought in marketSellEthForERC20 is the ZRX token. In this scenario the caller can supply an empty fee orders and fee signatures array.

### Fee Proportion and Fee Recipient

The Fee proportion and fee receipient are a feature in the Forwarding contract which is distinct from the ZRX fees on the orders. This incentivises integrations from wallets and other dapps who are not the feeRecipients in these orders to have a possible revenue stream. The feeProportion is calculated off the amount traded, not the total amount of Ether sent to the contract when calling the function.

Fee Proportion supports up to 2 decimial places, for example 0.59% is represented as 59. This value is limited to less than or equal to 10% or 1000.

### Acceptable Threshold

In order to avoid excessively high fees during the fee abstraction a threshold is introduced. This is proportional to the total amount traded (not the total amount of Ether sent). This prevents a disproporational amount of fees to the value traded from fees on both orders and feeOrders.

# Miscellaneous

## Optimizing calldata

Calldata is expensive. As per Appendix G of the [Ethereum Yellowpaper](#https://ethereum.github.io/yellowpaper/paper.pdf), every non-zero byte of calldata costs 68 gas, and every zero byte costs 4 gas. There are certain off-chain optimizations that can be made in order to maximize the amount of zeroes included in calldata.

### Assuming order parameters

The Forwarding contract assumes all takerAssetData is the WETH asset data and fills this in. This means users may pass in zero values for those parameters and the functions will still execute as if the values had been passed in as calldata. This applies to makerAssetData in fee abstraction (feeOrders), this data is filled in as ZRX/WETH.

# Events

There are no additional events emitted by the Forwarding contract. Token Transfer, WETH Deposit and Withdraw and 0x Exchange Fill are emitted in their respective contracts.

## Known Issues

### Over buying ZRX

In order to avoid any rounding issues when performing ZRX fee abstraction, an additional 1\*10^-18 ZRX is purchased. In certain scenarios this is left behind in the contract. It is unlikely this would ever accrue to a value worth extracting and is not worth the gas in transferring per trade.

When calculating the the expected fill results, in some scenarios it is possible to over buy ZRX fees. This is observable in the `marketSellEthForERC20` case as we calculate an expected fill given the total amount of Ether sent in (totalEth). Since fees are purchased, and Ether is spent (ethSpentFees) only totalEth-ethSpentFees is remaining to buy the tokens. This can result in a slight over purchase of fees by a slight amount. The cost of storing this amount for the user, plus withdrawal gas cost was deemed more expensive than the value of tokens over bought. The end result is the contract locking up this over bought ZRX.

### Fee Proportion and Fee Recipient

These values are submitted by the user and can therefor be modified. After talking with many projects we believe this is acceptable for multiple reasons.

1.  It is possible for a wallet to copy this contract and hard code these values.
2.  If a wallet is the source of liquidity, when creating or accepting orders they can specify the senderAddress.
3.  It is cheaper for an "attacker" to avoid the Forwarding contract entirely and transact directly with the 0x Exchange.
