# Dydx Orders
Dydx orders allow a maker to trade borrowed ERC20 tokens from a Dydx account they own. In the same trade, taker tokens can also be deposited into the maker's Dydx account as collateral. From the taker's perspective, filling these orders is no different from filling a normal ERC20->ERC20 order.

## Settlement
The taker side is settled through the regular `ERC20Proxy`. The maker side of Dydx orders are settled through the `ERC20BridgeProxy`, which calls the `DydxBridge` contract.

The simplified settlment process is:
1. The `ERC20Proxy` transfers taker tokens from the taker to the *maker*.
2. The `ERC20BridgeProxy` calls the `DydxBridge` which in turn calls [`SoloMargin`](https://etherscan.io/address/0x1e0447b19bb6ecfdae1e4ae1694b0c3659614e4e) (Dydx) to:
    1. Transfer taker tokens from the *maker* into their Dydx account (deposit).
    2. Borrow maker tokens from the *maker's Dydx account* to the *taker* (withdraw).

![settlement-diagram](https://raw.githubusercontent.com/0xProject/0x-protocol-specification/feat/dydx-bridge-orders/order-types/img/dydx-settlement.png)

## Required Approvals
- Makers must approve the Dydx [`SoloMargin`](https://etherscan.io/address/0x1e0447b19bb6ecfdae1e4ae1694b0c3659614e4e) contract to spend any *taker* tokens they intend to deposit into their account.
- Makers must add the [`DydxBridge`](https://etherscan.io/address/0x55dc8f21d20d4c6ed3c82916a438a413ca68e335) contract as an operator for their Dydx account via the [`SoloMargin.setOperators()`](https://github.com/dydxprotocol/solo/blob/d3bdc38ddfd224628b6dcd03812e57b3b62f2c82/contracts/protocol/Permission.sol#L62) function.

## Creating An Order
Dydx orders must correctly specify the following order fields:

- `makerAddress`: The maker address (not the bridge address).
- `takerAssetData`: Standard ERC20 token asset data.
- `makerAssetData`: `DydxBridge` encoded asset data (See [Maker Asset Data](#maker-asset-data)).

### Maker Asset Data
Dydx orders are actually `ERC20BridgeProxy` orders, so they have nested encodings. The (`DydxBridge`) `bridgeData` is nested inside a `ERC20BridgeProxy` asset data.

```solidity
order.makerAssetData = abi.encodeWithSelector(
    // ERC20BridgeProxy selector: bytes4(keccack256("ERC20Bridge(address,address,bytes)"))
    0xdc1600f3,
    // Maker token address
    makerToken,
    // Address of the DydxBridge contract
    DYDX_BRIDGE_ADDRESS,
    // DydxBridge bridgeData
    abi.encode(
        // uint256[] (dydx account numbers)
        accountNumbers,
        // DydxBridgeAction[] (deposit/withdraw actions)
        actions
    ),
);
```

Where `DydxBridgeAction` is defined as:

```solidity
struct DydxBridgeAction {
    DydxBridgeActionType actionType;
    uint256 accountIdx;
    uint256 marketId;
    uint256 conversionRateNumerator;
    uint256 conversionRateDenominator;
}
```

And `DydxBridgeActionType` is:

```solidity
enum DydxBridgeActionType {
    Deposit,
    Withdraw
}
```

### Actions
Dydx orders can execute one or more actions during settlement. There are only two supported actions: `Deposit` and `Withdraw`. The maker asset data can contain zero or more `Deposit` actions followed by *exactly one* `Withdraw` action, for the maker token.

- `Deposit` actions will transfer tokens from the *maker* into the maker's Dydx account, as collateral. If this token is also the taker token, then at least some of it will be provided by the taker, because tokens are transferred from taker to maker first.
- `Withdraw` actions will borrow tokens from the maker's Dydx account and transfer them to the taker.

#### Account Indices
Each action has an `accountIdx` field, which is an index into the `accountNumbers` array. This will determine the maker's Dydx account the action operates on.

#### Market IDs
The `marketId` is the market ID for the token the action operates with. You can call [`SoloMargin.getMarketTokenAddress()`](https://github.com/dydxprotocol/solo/blob/d3bdc38ddfd224628b6dcd03812e57b3b62f2c82/contracts/protocol/Getters.sol#L158) to discover what market IDs correspond to which tokens. For convenience, here are  known market IDs at the time of this writing:

| Market ID | Token Name | Token Address |
|-----------|------------|---------------|
|`0`        | WETH   | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` |
|`1`        | SAI    | `0x89d24A6b4CcB1B6fAA2625fE562bDD9a23260359` |
|`2`        | USDC   | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` |
|`3`        | DAI    | `0x6B175474E89094C44Da98b954EedeAC495271d0F` |


#### Rates
Action rates are specified by two values: a `conversionRateNumerator` and `conversionRateDenominator`. These two values define a fraction which is multiplied by the maker asset amount to be transferred from the maker to the taker. This product is the actual amount the action will deposit or withdraw:

```solidity
uint256 actionAmount =
    makerAssetAmountToTransfer
    * action.conversionRateNumerator
    / action.conversionRateDenominator;
```

##### Deposit Rates
This determines the rate at which tokens are deposited from the *maker* into a maker's Dydx account. It's important to remember that this rate will be scaled by the *maker asset amount* being transferred during settlement. So this rate should also convert the maker units into the units of the token being transferred.

For example, if a maker wants to deposit `0.5` WETH for every `1` maker token exchanged, they could compute the rate as:

```solidity
conversionRateNumerator = 0.5 * (10 ** WETH_DECIMALS);
conversionRateDenominator = (10 ** MAKER_DECIMALS);
```

To deposit all the tokens received from the taker, one can simply set these values to:

```solidity
conversionRateNumerator = order.takerAssetAmount;
conversionRateDenominator = order.makerAssetAmount;
```

##### Withdraw Rates
This determines the rate at which tokens are borrowed from a maker's Dydx account. Currently, only borrowing the maker token is supported.

Again, these rates will be scaled by the *maker asset amount* being transferred during settlement. So to borrow the full amount of maker tokens to transfer to the taker, you can set these values to identity:

```solidity
conversionRateNumerator = 1;
conversionRateDenominator = 1;
// or, this is an alias for 1/1:
conversionRateNumerator = 0;
conversionRateDenominator = 0;
```

### Example
Suppose a maker wants to accept WETH and return USDC. They want to deposit *half* the WETH they receive into their Dydx account (#1337) and keep the rest. They could build an order as follows:

```js
order = {
    ...OTHER_ORDER_FIELDS,
    makerAddress: MAKER_ADDRESS,
    takerAssetAmount: TAKER_ASSET_AMOUNT,
    makerAssetAmount: MAKER_ASSET_AMOUNT,
    takerAssetData: abi.encodeWithSelector(
        // ERC20Proxy selector
        0xf47261b0,
        // Taker token
        WETH_ADDRESS
    ),
    makerAssetData: abi.encodeWithSelector(
        // ERC20BridgeProxy selector
        0xdc1600f3,
        // Maker token
        USDC_ADDRESS,
        // Address of the DydxBridge contract.
        DYDX_BRIDGE_ADDRESS,
        // Bridge data
        abi.encode(
            // Dydx Account numbers
            [ 1337 ],
            [
                // Deposit action
                {
                    accountType: DydxBridgeActionType.Deposit,
                    // Account idx 0: 1337
                    accountIdx: 0,
                    // Market ID 0: WETH
                    marketId: 0,
                    // Deposit half the WETH received
                    conversionRateNumerator: 0.5 * TAKER_ASSET_AMOUNT,
                    conversionRateDenominator: MAKER_ASSET_AMOUNT
                },
                // Withdraw action
                {
                    accountType: DydxBridgeActionType.Withdraw,
                    // Account idx 0: 1337
                    accountIdx: 0,
                    // Market ID 0: WETH
                    marketId: 0,
                    conversionRateNumerator: 1,
                    conversionRateDenominator: 1
                }
            ]
        )
    )
}

```

## Benchmarks

As expected, Dydx orders consume more gas. This gas cost scales with every action.
- 1 withdraw action: `~315k`
- 1 withdraw and 1 deposit action: `~375k`
