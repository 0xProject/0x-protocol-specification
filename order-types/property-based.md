# Property-based Orders
One of the asset types supported by the 0x Exchange are non-fungible ERC721 tokens, commonly used for collectibles and game items.
To create a [regular 0x order]((https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#orders)) buying an [ERC721 asset](https://github.com/0xProject/0x-protocol-specification/blob/master/asset-proxy/erc721-proxy.md), one would need to specify exactly which token to buy.
Property-based orders, on the other hand, can be created to buy *any* asset that satisfies the properties specified by the maker of the order.

## Broker contract
Property-based orders cannot be filled directly via the Exchange contract because the taker asset is not fully determined until the fill.
Instead, we introduce the [Broker](https://github.com/0xProject/ZEIPs/issues/75) architecture to serve as the taker's entry-point. Refer to the linked ZEIP for motivation and technical details of the contract architecture.

The Broker contract exposes two entry-points:
```solidity
/// @dev Fills a single property-based order by the given amount using the given assets.
///      Pays protocol fees using either the ETH supplied by the taker to the transaction or
///      WETH acquired from the maker during settlement. The final WETH balance is sent to the taker.
/// @param brokeredTokenIds Token IDs specified by the taker to be used to fill the orders.
/// @param order The property-based order to fill. The format of a property-based order is the
///        same as that of a normal order, except the takerAssetData. Instaed of specifying a
///        specific ERC721 asset, the takerAssetData should be ERC1155 assetData where the
///        underlying tokenAddress is this contract's address and the desired properties are
///        encoded in the extra data field. Also note that takerFees must be denominated in
///        WETH (or zero).
/// @param takerAssetFillAmount The amount to fill the order by.
/// @param signature The maker's signature of the given order.
/// @param fillFunctionSelector The selector for either `fillOrder` or `fillOrKillOrder`.
/// @param ethFeeAmounts Amounts of ETH, denominated in Wei, that are paid to corresponding feeRecipients.
/// @param feeRecipients Addresses that will receive ETH when orders are filled.
/// @return fillResults Amounts filled and fees paid by the maker and taker.
function brokerTrade(
    uint256[] calldata brokeredTokenIds,
    LibOrder.Order calldata order,
    uint256 takerAssetFillAmount,
    bytes calldata signature,
    bytes4 fillFunctionSelector,
    uint256[] calldata ethFeeAmounts,
    address payable[] calldata feeRecipients
)
    external
    payable
    returns (LibFillResults.FillResults memory fillResults);

/// @dev Fills multiple property-based orders by the given amounts using the given assets.
///      Pays protocol fees using either the ETH supplied by the taker to the transaction or
///      WETH acquired from the maker during settlement. The final WETH balance is sent to the taker.
/// @param brokeredTokenIds Token IDs specified by the taker to be used to fill the orders.
/// @param orders The property-based orders to fill. The format of a property-based order is the
///        same as that of a normal order, except the takerAssetData. Instaed of specifying a
///        specific ERC721 asset, the takerAssetData should be ERC1155 assetData where the
///        underlying tokenAddress is this contract's address and the desired properties are
///        encoded in the extra data field. Also note that takerFees must be denominated in
///        WETH (or zero).
/// @param takerAssetFillAmounts The amounts to fill the orders by.
/// @param signatures The makers' signatures for the given orders.
/// @param batchFillFunctionSelector The selector for either `batchFillOrders`,
///        `batchFillOrKillOrders`, or `batchFillOrdersNoThrow`.
/// @param ethFeeAmounts Amounts of ETH, denominated in Wei, that are paid to corresponding feeRecipients.
/// @param feeRecipients Addresses that will receive ETH when orders are filled.
/// @return fillResults Amounts filled and fees paid by the makers and taker.
function batchBrokerTrade(
    uint256[] calldata brokeredTokenIds,
    LibOrder.Order[] calldata orders,
    uint256[] calldata takerAssetFillAmounts,
    bytes[] calldata signatures,
    bytes4 batchFillFunctionSelector,
    uint256[] calldata ethFeeAmounts,
    address payable[] calldata feeRecipients
)
    external
    payable
    returns (LibFillResults.FillResults[] memory fillResults);
```

## Creating a property-based order
To create a property-based 0x order, we need:
- The address of the `Broker` contract, which can be found in the [`@0x/contract-addresses` package](https://www.npmjs.com/package/@0x/contract-addresses).
- The properties we'd like the taker asset to satisfy.
- A property validator contract, which the Broker will `staticcall` to verify that the taker-supplied asset satisfies the specified properties. The validator must implement the [`IPropertyValidator` interface](#writing-a-property-validator-contract).
- Any additional properties required by the validator.

All these are then encoded into [ERC1155 asset data](https://github.com/0xProject/0x-protocol-specification/blob/master/asset-proxy/erc1155-proxy.md) interpretable by the Broker. Consider the following TypeScript example, which performs the encoding for [Gods Unchained cards](https://github.com/immutable/platform-contracts/blob/master/contracts/gods-unchained/contracts/Cards.sol) and the [`GodsUnchainedValidator` contract](https://github.com/0xProject/0x-monorepo/blob/development/contracts/broker/contracts/src/validators/GodsUnchainedValidator.sol).
```typescript
/**
 * Encodes the given proto and quality into ERC1155 asset data to be used as the takerAssetData
 * of a property-based GodsUnchained order. Must also provide the addresses of the Broker,
 * GodsUnchained, and GodsUnchainedValidator contracts. The optional bundleSize parameter specifies
 * how many cards are expected for each "unit" of the takerAssetAmount. For example, If the
 * takerAssetAmount is 3 and the bundleSize is 2, the taker must provide 2, 4, or 6 cards
 * with the given proto and quality to fill the order. If an odd number is provided, the fill fails.
 */
encodeBrokerAssetData(
    brokerAddress: string,
    godsUnchainedAddress: string,
    validatorAddress: string,
    proto: BigNumber,
    quality: BigNumber,
    bundleSize: number = 1,
): string {
    const dataEncoder = AbiEncoder.create([
        { name: 'godsUnchainedAddress', type: 'address' },
        { name: 'validatorAddress', type: 'address' },
        { name: 'propertyData', type: 'bytes' },
    ]);
    const propertyData = AbiEncoder.create([
        { name: 'proto', type: 'uint16' },
        { name: 'quality', type: 'uint8' },
    ]).encode({ proto, quality });
    const data = dataEncoder.encode({ godsUnchainedAddress, validatorAddress, propertyData });
    return assetDataUtils.encodeERC1155AssetData(brokerAddress, [], [new BigNumber(bundleSize)], data);
}
```
[\[Source\]](https://github.com/0xProject/0x-monorepo/blob/development/contracts/broker/src/gods_unchained_utils.ts)

The above asset data can also be nested inside [MultiAsset](https://github.com/0xProject/0x-protocol-specification/blob/master/asset-proxy/multi-asset-proxy.md) asset data, so that one can create an order buying a "bundle" of property-satisfying assets.

## Filling a property-based order
Property-based orders must be filled via the [Broker](https://github.com/0xProject/ZEIPs/issues/75).
The taker must set allowances enabling the Broker to trade the brokered assets on their behalf.

Orders filled through the Broker must satisfy the following properties:
- The `takerAssetData` is a **brokered asset**, i.e. ERC1155 asset data in the format expected by the Broker; or it is WETH asset data.
- The taker fee is either zero or denominated in WETH.
In order to pay [protocol fees](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#protocol-fees) and fill orders with the taker asset or fee denominated in WETH, the taker must send sufficient ETH with their `brokerTrade`/`batchBrokerTrade` transaction. Any remaining ETH at the end of the transaction is refunded.

If the order[s] provided to `brokerTrade` or `batchBrokerTrade` involve _multiple_ brokered assets, the taker must supply that many tokens in the `brokeredTokenIds` parameter.
Those tokens will be used sequentially (in the order provided) every time a brokered asset is required.

### Errors
In addition to [errors](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#exchange-errors) that can be thrown by the Exchange, the Broker itself can throw the following errors. The Broker will also rethrow any errors thrown by a property validator contract.

| Error                                                       | Condition                                                                                        |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| OnlyERC1155ProxyError                                       | An address other than the ERC1155AssetProxy calls `safeBatchTransferFrom`                        |
| InvalidFromAddressError                                     | An order specified brokered assets in its `makerAssetData` or `makerFeeAssetData`                |
| AmountsLengthMustEqualOneError                              | Only one asset amount should be specified in the ERC1155-encoded broker asset data               |
| TooFewBrokerAssetsProvidedError                             | Not enough token IDs supplied by the broker to perform the fill                                  |

These will normally be caught in the Exchange and rethrown as an [`AssetProxyTransferError`](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#assetproxytransfererror), where the nested `errorData` encodes the underlying Broker error.

## Writing a property validator contract
You can implement and deploy your own property validator contract.
To be compatible with the Broker contract, it should implement the following interface:
```solidity
interface IPropertyValidator {

    /// @dev Checks that the given asset data satisfies the properties encoded in `propertyData`.
    ///      Should revert if the asset does not satisfy the specified properties.
    /// @param tokenId The ERC721 tokenId of the asset to check.
    /// @param propertyData Encoded properties or auxiliary data needed to perform the check.
    function checkBrokerAsset(
        uint256 tokenId,
        bytes calldata propertyData
    )
        external
        view;
}
```
For an example of TypeScript tooling for a property validator, refer to [`gods_unchained_utils.ts`](https://github.com/0xProject/0x-monorepo/blob/development/contracts/broker/src/gods_unchained_utils.ts).
