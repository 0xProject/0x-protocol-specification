# Stop-limit Orders
[Stop-limit orders](https://www.investopedia.com/terms/s/stop-limitorder.asp) are orders that can only be filled if the market price of the token pair being traded (as reported by some oracle) is within a certain range.
For background on the 0x order format, refer to the [v3 specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders).

## Chainlink stop-limit contract
Stop-limit 0x orders are enabled by the `ChainlinkStopLimit.sol` contract, used in conjunction with the [StaticCall](https://github.com/0xProject/0x-protocol-specification/blob/master/asset-proxy/static-call-proxy.md) asset proxy.
This contract exposes a single function, `checkStopLimit`, which checks the value returned by a given Chainlink reference contract and reverts if it is not within the specified range.

```solidity
contract ChainlinkStopLimit {

    /// @dev Checks that the price returned by the encoded Chainlink reference contract is
    ///      within the encoded price range.
    /// @param stopLimitData Encodes the address of the Chainlink reference contract and the
    ///        valid price range.
    function checkStopLimit(bytes calldata stopLimitData)
        external
        view;
}
```

## Creating a stop-limit order
To create a stop-limit 0x order, we need:
- The address of the `ChainlinkStopLimit` contract, which can be found in the [`@0x/contract-addresses` package](https://www.npmjs.com/package/@0x/contract-addresses).
- The address of the Chainlink contract for the pair whose price we'd like to create the stop-limit around. A list of those contracts can be found [here](https://feeds.chain.link/).
- The price range in which we'd like the order to be executed. Note that the value returned by Chainlink reference contracts is the price multiplied by 100000000 ([source](https://docs.chain.link/docs/using-chainlink-reference-contracts#section-live-reference-data-contracts-ethereum-mainnet)).

The contract addresses and parameters must be encoded as StaticCall asset data and included in one of the order's asset data fields.
For example, consider an order selling ZRX for ETH.

### Option 1: makerFeeAssetData

If the `makerFeeAssetData` would otherwise be unused, we can simply set it to be the stop-limit StaticCall asset data.
During order settlement, the Exchange will dispatch the StaticCall proxy to call the stop-limit contract.

In TypeScript, this would look something like the following:
```typescript
import { encodeStopLimitStaticCallData } from '@0x/contracts-integrations';
import { assetDataUtils } from '@0x/order-utils';

const makerFeeAssetData = encodeStopLimitStaticCallData(
    chainlinkStopLimit.address,
    chainLinkZrxEthAggregator.address,
    minPrice,
    maxPrice,
);

```

In order to avoid unexpected behavior caused by rounding and overflow, we recommend setting the `makerFee` amount to equal the `makerAssetAmount` (in this case, the amount of ZRX being sold).
Note that despite the non-zero `makerFee`, the maker will not be transferring any assets for the "transfer", we are simply repurposing the field to perform the stop-limit check.
As such, the `feeRecipientAddress` can be set to an arbitrary address or the null address.

### Option 2: Multi-asset makerAssetData

Option 1 is the preferred way to create stop-limit orders for easy discoverability via [0x Mesh](https://github.com/0xProject/0x-mesh).
However, if the maker would in fact like to use the maker fee field to transfer assets to a fee recipient, we can instead add the stop-limit check to the `makerAssetData`.

Normally, the `makerAssetData` of a ZRX-ETH sell order would be the ERC20 asset data encoding of the ZRX token address.
But to turn this into a stop-limit sell order, we can instead use [MultiAsset](https://github.com/0xProject/0x-protocol-specification/blob/master/asset-proxy/multi-asset-proxy.md) asset data, where the nested assets are ZRX and the stop-limit check (encoded as StaticCall asset data).
During order settlement, the MultiAsset proxy will dispatch the ERC20 proxy to perform the ZRX transfer and then dispatch the StaticCall proxy to call the stop-limit contract.

In TypeScript, this would look something like the following:
```typescript
import { encodeStopLimitStaticCallData } from '@0x/contracts-integrations';
import { assetDataUtils } from '@0x/order-utils';

const makerAssetData = assetDataUtils.encodeMultiAssetData(
    [new BigNumber(100), new BigNumber(1)],
    [
        assetDataUtils.encodeERC20AssetData(zrxToken.address),
        encodeStopLimitStaticCallData(
            chainlinkStopLimit.address,
            chainLinkZrxEthAggregator.address,
            minPrice,
            maxPrice,
        ),
    ],
);
```

## Filling a stop-limit order
Stop-limit orders can be filled directly via the [Exchange](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#exchange) and do not require additional allowances. One can check whether an order is fillable at the current price by calling the `ChainlinkStopLimit` contract's `checkStopLimit` function.

### Errors
If the stop-limit price check fails, the transaction will revert and throw an [`AssetProxyTransferError`](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#assetproxytransfererror), where the nested `errorData` encodes the error thrown by the stop-limit contract: `"ChainlinkStopLimit/OUT_OF_PRICE_RANGE"`.

## Writing a stop-limit validator contract
Currently, the only on-chain oracles supported by 0x stop-limit orders are Chainlink reference contracts.
That said, you can implement and deploy a validator contract with your oracle of choice.
Refer to the [Chainlink stop-limit contract](https://github.com/0xProject/0x-monorepo/blob/development/contracts/integrations/contracts/src/ChainlinkStopLimit.sol) and [TypeScript tooling](https://github.com/0xProject/0x-monorepo/blob/development/contracts/integrations/src/chainlink_utils.ts) for how to do so.
