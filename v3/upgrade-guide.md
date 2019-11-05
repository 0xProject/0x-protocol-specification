# V3 Upgrade Guide

The core 0x team have been hard at work the past year to add new features to the 0x protocol. There are a number of exciting changes and below we will walk through the most important changes for projects in the ecosystem.

For a full list of differences in the protocol refer to the [changelog here](https://github.com/0xProject/0x-protocol-specification/blob/3.0/v3/v3-specification.md#differences-from-20.).

### Code Examples

[0x Starter Project](https://github.com/0xProject/0x-starter-project/tree/3.0)

# Important information by use case

## As a Market Maker

* Sharing liquidity via Mesh is the fastest and widest distribution channel to reach prospective takers. Deploy your own Mesh node and subscribe to events to allow you to discover other orders and receive fill updates on your orders.  Mesh works with 0x v2, so it is good idea to become acquainted with Mesh and adopt it into your architecture before migrating to v3. [Mesh And Networked Liquidity](#mesh-and-networked-liquidity)
* When transitioning to v3, the only required action is to cancel your v2 orders and create new orders on v3. Orders from v2 cannot be reposted to v3 as there is replay protection. Cancellation can be performed efficiently using `cancelOrdersUpTo` (if using an incrementing salt) or can be performed order by order with `batchCancelOrders`. Ensure your orders are cancelled on v2 prior to creating new orders on v3. 
* Upgrade to the latest 0x packages using the `protocolV3` tag. For example,  `yarn upgrade @0x/order-utils@protocolV3`
* The exchange address and order format changes from v2 to v3, see [Order Format](#order-format). Mainnet Exchange address is TBD.
* As part of the 0x governance upgrade, the ERC20 approvals set on the 0x Asset Proxies will be migrated to v3. So no additional approvals are required. 

* You will begin to receive rebates from takers that fill your orders, see [Staking And Rebates](#staking-and-rebates) for more information. The 0x team will be reaching out to known Market Makers to assist in setup.

## Taker/CFL

* The ecosystem is moving to using 0x Mesh to share and discover liquidity. 0x Mesh is a decentralized network where all participants share orders equally with each other, see [Mesh And Networked Liquidity](#mesh-and-networked-liquidity). We recommend projects deploy their own Mesh nodes to contribute to the network. For light clients we have made gateway to Mesh [available as an endpoint](http://sra.0x.org/v3/orders) which follows the [SRA specification.](https://github.com/0xProject/standard-relayer-api)
* Upgrade to the latest 0x packages using the `protocolV3` tag. For example,  `yarn upgrade @0x/order-utils@protocolV3`
* For Takers that Fill orders via a Contract
    * Takers now pay a [Protocol Fee](#protocol-fee) which is ultimately rebated to Market Makers. The entrypoint function used may need to become payable in order to forward the fee from `msg.sender`. This fee can also be paid in WETH with an additional allowance.
    * The 0x [Order Format](#order-format) has changed with new additional fields. Ensure your contracts are updated to the new Order struct.
    * As part of the 0x governance upgrade, the ERC20 approvals set on the 0x Asset Proxies will be migrated to v3. So no additional approvals are required. 
* For Takers that Fill orders via an Externally Owned Account
    * Takers now pay a [Protocol Fee](#protocol-fee) which is ultimately rebated to Market Makers. This fee is paid by attaching ETH to the transaction.
    * As part of the 0x governance upgrade, the ERC20 approvals set on the 0x Asset Proxies will be migrated to v3. So no additional approvals are required. 
* As part of the 0x governance upgrade, the ERC20 approvals set on the 0x Asset Proxies will be migrated to v3. So no additional approvals are required
* The Exchange contract is new and has a different Exchange address

## Relayer

* The ecosystem is moving to using 0x Mesh to share and discover liquidity. 0x Mesh is a decentralized network where all participants share orders equally with each other, see [Mesh And Networked Liquidity](#mesh-and-networked-liquidity). We recommend projects deploy their own Mesh nodes to contribute to the network. 
* Upgrade to the latest 0x packages using the `protocolV3` tag. For example,  `yarn upgrade @0x/order-utils@protocolV3`
* Follow along the [Important Dates](https://github.com/0xProject/ZEIPs/issues/56) section to know the exact dates milestones are occurring
* Inform your users of the upcoming v2 to v3 upgrade
* Allow your users to cancel v2 orders and create new orders for v3. The exchange address will be known ahead of time, so orders can be created prior to the protocol being upgraded.
* [Protocol Fee](#protocol-fee)s will be paid by takers of 0x liquidity, educate your users on this process. It is roughly equivalent to the gas fee already being paid to the miner. Your users will see an increase in the transaction cost in ETH.
* Become acquainted with the calculation of the protocol fee, `150,000 * ordersFilled * gasPrice`. If your users **lower** their gas price in the wallet the protocol will return the excess fees. If your users **increase **the gas price it is possible the provided ETH value in the transaction is not enough to cover the Protocol Fee.

# Concepts

### Mesh and networked liquidity

Mesh is our latest evolution of networked liquidity. Prior to Mesh, users were reliant on a centralized order store connecting the two parties in order to share and discover orders. This centralised order store was instrumental to early success and adoption of 0x but also introduced problems. Often times the order store had out of date orders or expired orders, increasing the likelihood of transaction failures and reducing the trust of the order store. Often takers would be required to re-validate all of the orders returned from the order store. These order stores did not have access to all of the available liquidity due to developer overhead in interacting with all other order stores. Order Stores also required whitelisting of certain tokens, exhibiting the same problems of centralized exchanges.

We believe in creating an open protocol where value can flow freely. There should not be a single, centralized place where liquidity for certain pairs is available. As such, Mesh uses a peer to peer network to share orders in a decentralised network. Access to this network is open and orders are shared throughout the network. This improves the distribution of orders and to all users without a centralized party controlling access. In the past a maker published their orders to a particular Relayer using `@0x/connect`. Going forward Mesh is the best distribution channel for orders to the greater ecosystem. 

Receiving timely updates when orders are filled, cancelled or expired is critical to operational success for a professional market maker. From knowing the state of their open positions, to keeping an up-to-date model of available orders, fast and quick updates to on-chain activity are critical. Mesh provides optimized state watching functionality. If in the past you have used our `@0x/order-watcher` package, this is a significant improvement and is a replacement for OrderWatcher. 

For more information on Mesh refer to [the documentation](https://0x-org.gitbook.io/mesh/) and watch the [video](https://www.youtube.com/watch?v=CIfQUW4oi-g).

The 0x team will provide a queryable Gateway over HTTP to Mesh to ease implementation. This can be used by takers or any other projects interested in discovering orders in the Mesh network.

Docker image for Mesh using V3 can be found [here](https://hub.docker.com/layers/0xorg/mesh/0xV3/images/sha256-8515b9a5a8b8f4c1e38048ea5cd81e03a3a2a8ae4129c5852433651fc4029c80).

```bash
docker pull 0xorg/mesh:0xv3
```

### Order Format

With the change to support arbitrary fee tokens (see [Arbitrary Fee Token](#arbitrary-fee-token)) there are two additional fields in the Order structure. They are the maker fee asset and the taker fee asset, encoded as Asset Data.

```solidity
Order(
  address makerAddress,
  address takerAddress,
  address feeRecipientAddress,
  address senderAddress,
  uint256 makerAssetAmount,
  uint256 takerAssetAmount,
  uint256 makerFee, // Changed: fee in maker fee asset (previously ZRX)
  uint256 takerFee, // Changed: fee in taker fee asset (previously ZRX)
  uint256 expirationTimeSeconds,
  uint256 salt,
  bytes makerAssetData,
  bytes takerAssetData,
  bytes makerFeeAssetData, // New: field added
  bytes takerFeeAssetData  // New: field added
)
```

### Staking and rebates

More information to come on Staking and Rebates. For now you tune into the [video explaining the changes](https://www.youtube.com/watch?v=s2wlzlQxd5E).

For more information on the Staking and Rebates refer to the [specification](https://github.com/0xProject/0x-protocol-specification/blob/3.0/v3/v3-specification.md#staking).

### Accessing on-chain liquidity

0x v3 now has the ability to access sources of on-chain liquidity. This enables developers and their users to use a single interface to tap into both off-chain and on-chain liquidity. A developer can choose which model works best for them and over time optimize and lower spread for their users with minimal code changes. For example, orders filled by a taker on 0x can consist of native 0x orders as well as Kyber reserves.

In essence the Maker is a contract with arbitrary logic that is executed. 0x Protocol guarantees the takers balance increases by the specified amount. The settlement flows from the taker to the maker, maker contract is executed and transfers funds to the taker, the 0x Protocol enforces the user’s balance of the specified token increases by the expected amount.


```solidity
// 0x Protocol will call the withdrawTo function of the bridge contract
// This contract must transfer the tokens to the to address. 0x Protocol 
// will verify the to address has a balance increase or it will revert.
contract CustomERC20Bridge {
    function bridgeTransferFrom(
        address toTokenAddress,
        address /* from */,
        address to,
        uint256 amount,
        bytes calldata bridgeData
    )
        external
        returns (bytes4 success);
}
```


For more information on Accessing On-chain Liquidity refer to [ZEIP-47](https://github.com/0xProject/ZEIPs/issues/47).

Example On-chain bridges:
[Eth2Dai](https://github.com/0xProject/0x-monorepo/pull/2221/files#diff-c0caf70ac828a1b07a4f14faab441381R29)
[Uniswap](https://github.com/0xProject/0x-monorepo/pull/2233/files#diff-c216e27ef3bc70f4b7da08ebaadfa8ecR32)

### Protocol fee

To foster a sustainable ecosystem, v3 has introduced Protocol Fees. These fees are paid by the Taker and ultimately result in a rebate for the Maker. The Protocol Fee scales with the Gas Price used by the transaction. In the wild we have seen high gas prices used by arbitrage bots when there is an opportunity for arbitrage. We see that these arbitrage bots compete with each other, resulting in high gas prices and in this competition the miner captures the most value. With Protocol Fees, a portion of funds used in this gas price auction can be redirected from the Miner to the Maker. Over time Makers receive the rebate from the Protocol Fees and are able to lower their operational costs, thus resulting in better prices. This is our next step in creating a sustainable ecosystem which rewards the users of the protocol. 

Protocol Fees are paid by the Taker in ETH or in wETH and are calculated as follows: `150,000 * gasPrice * orders filled`. When filling via a contract, be sure to take the protocol fee into account as a payable function, or to ensure the contract has an available wETH balance and allowance set. 

The Fill operations in the protocol have been changed to payable functions. Takers trading from an Externally Owned Account (EOA) must send a sufficient amount of ETH to pay the protocol feel. Contracts which use 0x Liquidity need to ensure their implementation passes down the required ETH through the various contracts. Alternatively the contract which performs the fill operation can have a WETH balance and allowance set to pay the fee. It is safe to overfund the amount of ETH for the Protocol Fee, the excess will be returned.

```solidity
contract MyContract {

  public address zeroExExchangeAddress;

  function liquidityRequiringMethod(bytes calldata calldataHexString) payable {
    zeroExExchangeAddress.call(calldataHexString).value(msg.value);
  }
}
```

For more information on the Protocol Fee refer to the [documentation](https://github.com/0xProject/0x-protocol-specification/blob/3.0/v3/protocol-fees.pdf) and the [specification](https://github.com/0xProject/0x-protocol-specification/blob/3.0/v3/v3-specification.md#protocol-fees). See [Staking And Rebates](#staking-and-rebates) for additional background information on the reasoning behind the Protocol Fee.

The Protocol Fee can be paid in ETH on all of the fill functions, the ETH is sent in as the value to the transaction:

```typescript
txHash = await contractWrappers.exchange.fillOrder.sendTransactionAsync(
    signedOrder,
    takerAssetAmount,
    signedOrder.signature,
    {
        from: taker,
        value: Web3Wrapper.toBaseUnitAmount(0.0006, 18),
    },
);
```

The Protocol Fee can be also be paid in WETH, as such an ERC20 approval is required. The protocol forwent using the ERC20Proxy to collect fees. To pay the Protocol Fee using WETH the taker must approve the Protocol Fee Collector contract.

```typescript
// Retrieve the protocol fee collector address
const protocolFeeCollector = await contractWrappers.exchange.protocolFeeCollector.callAsync();
// Allow the protocol fee collector to use WETH as the protocol fee token
const takerWETHProtocolApprovalTxHash = await etherToken.approve.validateAndSendTransactionAsync(
    protocolFeeCollector,
    UNLIMITED_ALLOWANCE_IN_BASE_UNITS,
    { from: taker },
);
```

### Arbitrary fee token

In previous versions of the protocol the Fee Recipient (often the Relayer) received fees in ZRX token. The use of ZRX as the fee token was a blocker as the user experience of holding ZRX was not viable. A user selling a collectible for 100 wETH would be required to hold a percentage of ZRX to pay the Relayer on settlement. This collectible could take weeks or months to sell, and in a very volatile market. 

Matching Relayers were the most successful in building towards a sustainable business by taking a fee through spread. This model allows a Matching Relayer to only Match orders (against their own) when there was appropriate spread to at least cover the operational cost. The latest fee changes will reduce the complexity of a Matching Relayer and calculating the profitability of executing a trade and the fees received. 

The 0x protocol in v3 now allows for the fee to be paid in any arbitrary token. Fees can be paid to the Fee Recipient in the tokens being traded or in any other arbitrary token. Fees can even be paid in collectibles. See the updated [Order Format](#order-format) for the parameters on arbitrary fee tokens.

### Affiliate fees (Forwarder)

To encourage the proliferation of 0x liquidity throughout the ecosystem, we introduced Affiliate Fees via the Forwarder contract. This enabled third parties to build towards a sustainable model by offering up network liquidity on their websites and applications. A wallet was able to embed 0x liquidity and take an ETH based fee from the operation. This model also incentivised other Relayers to show orders to their users where they were not the Fee Recipient of the order. 

The latest version of the Forwarder has kept this functionality and we will continue to iterate on this model.

### Rich Revert reasons and Debugging

Having easy to use tools and documentation is fundamental to a great user experience. Having informative errors and insight to problems is arguably more important. A developer doesn’t always stay on the golden path, sometimes things go wrong. With v3 we have focused on propagating more information to the developer when operations go wrong. This includes reporting the particular order which caused the error as well as which attribute on the order caused the error. You will no longer see errors like `ORDER_UNFILLABLE` when performing a market sell operation with numerous orders. 

An example of a Rich Revert message:

```solidity
// Fill order on a cancelled order
bytes4(keccak256("OrderStatus(bytes32,uint8)")),
    orderHash,
    orderStatus
);
// Example data
0x03584e62f8ab825fac82a18bbd396384b38c46c96b4f8f84cb2776aad307dbe6 CANCELLED


// Invalid Balance of Allowance during settlement
bytes4(keccak256("AssetProxyTransferError(bytes32,bytes,bytes)")),
    orderHash,
    assetData,
    errorData,
);
// Example data
0x03584e62f8ab825fac82a18bbd396384b38c46c96b4f8f84cb2776aad307dbe6 0xf47261b0000000000000000000000000960b236a07cf122663c4303350609a66a7b288c0
```


These improvements are not siloed to the tools like the typescript packages, but built into the core contracts. So no matter which language or web3 library you use to interact with the 0x contracts, you will receive informative errors when things don’t go according to plan.

For more information on Rich Reverts refer to the [documentation here](https://github.com/0xProject/0x-protocol-specification/blob/3.0/v3/v3-specification.md#rich-reverts).

### Support for more languages

The diversity of programming languages being used by the ecosystem is remarkable. In fact, it can be overwhelming for a small team to create a package for every programming language being used. As a result we weren’t able to support all languages with the same degree as we do for Javascript and Typescript. To enable greater participation from the ecosystem we have begun to move core functionality to on-chain contracts. These contracts can be used for common operations involving 0x, like hashing orders, encoding asset data and checking order validity. Since the contracts exist on-chain they can be called from any web3 library using an `eth_call`. This reduces the maintenance overhead of adding new features and reduces inconsistencies across languages as the core functionality is shared. Our very own 0x Mesh utilises these core modules and it has resulted in huge performance gains over multiple `eth_call` to perform the same operations (like checking user balances and allowances).

DevUtils allows you to perform the following with only a web3 package:

* Encoding and Decoding Asset Data
* Validation of Orders, Order Status and Signatures


Our primary on-chain core module is called DevUtils and [more information can be found here](https://etherscan.io/address/0x92d9a4d50190ae04e03914db2ee650124af844e6#readContract).

### Market Buy and Sell Operations

The 0x Protocol supports Market Sell and Market Buy Operations. Given a set of orders, the protocol will iterate through each Order until the requested amount has been settled, or all orders have been exhausted. There were two variants of this functionality, for Market Buy there was marketBuyOrders and marketBuyOrdersNoThrow. marketBuyOrders behaved in undesirable way when it encountered an order which had been cancelled or expired. When an unfillable order was encountered it would revert rather than move to the continue to the next order. Most implementations moved to using `markeyBuyOrdersNoThrow` to avoid this behavior. 

A new operation, `marketBuyOrdersFillOrKill` has been introduced in v3. This function only reverts if the target amount of tokens are unable to be bought or sold, if an unfillable order is encountered it is skipped. This function is useful as it ensures the user get’s the requested amount of tokens when buying. If 500 DAI is required to close a CDP, this function will ensure the user receives at least 500 DAI. 

### Order Cancellations

Makers often need to cancel orders. We’ve seen a number of Makers use `cancelOrdersUpTo` which can cancel an arbitrary number of orders by invalidating a salt value. Any orders with a salt value less than this are considered invalid.

Those who use cancelOrder or batchCancelOrders may have experienced frustration as this function was not idempotent. If a previously cancelled order was included in a batchCancelOrders, then the operation would revert. In v3 this behavior has changed and the operation is idempotent. A Maker is now able to include previously cancelled and expired orders in the `cancelOrder` and `batchCancelOrders` operations.

### Standard Relayer API

0x Released a specification for a Standard Relayer API (SRA). The primary purpose of this was to enable Relayers to share orders with other Relayers, and allow other participants in the ecosystem to discover orders programatically. With the release of 0x Mesh ([Mesh And Networked Liquidity](#mesh-and-networked-liquidity)), this has replaced the need for Relayers to share orders with other Relayers as now all orders can be shared in a decentralized peer-to-peer network. In the interim we have updated the SRA to support v3 with the new [Order Format](#order-format). User facing tools such as AssetSwapper and 0x Instant will still rely on the SRA to discover orders. 

The final spec for SRA v3 is [complete](https://github.com/0xProject/standard-relayer-api#sra-v3). 

### Contract Addresses

Each test network will have new contracts deployed. Please refer to the network by ID in the [this file](https://github.com/0xProject/0x-monorepo/blob/3.0/packages/contract-addresses/src/index.ts#L84) to find the contract address you need. The `@0x/contract-addresses` package can also be imported to access the addresses programatically.

```
Mainnet
Exchange: TBD
Forwarder: TBD

Kovan
Exchange: 0x617602cd3f734cf1e028c96b3f54c0489bed8022
Forwarder: 0x4c4edb103a6570fa4b58a309d7ff527b7d9f7cf2

Ropsten
Exchange: 0x725bc2f8c85ed0289d3da79cde3125d33fc1d7e6
Forwarder: 0x31c3890769ed3bb30b2781fd238a5bb7ecfeb7c8 

Rinkeby
Exchange: 0x8e1dfaf747b804d041adaed79d68dcef85b8de85
Forwarder: 0xc6db36aeb96a2eb52079c342c3a980c83dea8e3c
```


