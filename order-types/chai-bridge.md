# ChaiBridge Orders

`ChaiBridge` orders allow market makers to earn the [Dai Savings Rate](https://community-development.makerdao.com/makerdao-mcd-faqs/faqs/dsr) while providing liquidity in Dai. This uses a combination of the [Chai token contract](https://chai.money/about.html) and the [ERC20BridgeProxy](../asset-proxy/erc20-bridge-proxy.md#erc20bridgeproxy) to create a user experience that is comparable to filling regular Dai-denominated orders.

## ChaiBridge contract

The `ChaiBridge` contract is a thin adapter between the Chai token contract and the `ERC20BridgeProxy` contract. Calling `ChaiBridge.bridgeTransferFrom` will perform the following steps:

1. Ensure that the sender is the `ERC20BridgeProxy`
1. Withdraw `amount` of Dai from the maker's (`from` address) Chai token balance
1. Transfer `amount` of Dai to the taker (`to` address)

This allows the `ERC20BridgeProxy` to transfer Dai from the maker's address to the taker's address, even though the maker is actually only holding Chai tokens (which are earning the Dai Savings Rate).

## Errors

The `ChaiBridge` contract's `bridgeTransferFrom` method may revert with the following errors:

| Error                                                                                                       | Condition                                                                                                           |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| [StandardError("ChaiBridge/ONLY_CALLABLE_BY_ERC20_BRIDGE_PROXY")](../v3/v3-specification.md#standard-error) | `bridgeTransferFrom` was called by an address other than the `ERC20BridgeProxy`                                     |
| [StandardError("ChaiBridge/DRAW_DAI_FAILED")](../v3/v3-specification.md#standard-error)                     | The `ChaiBridge` was not able to withdraw the specified `amount` of Dai from the maker's Chai tokens for any reason |

## Setup for ChaiBridge orders

In order for a maker to begin providing Dai liquidity through the `ChaiBridge` contract, the maker must first convert their Dai to Chai and then approve the `ChaiBridge` contract to spend their Chai tokens.

```typescript
const MAX_UINT256 = new BigNumber(2).pow(256).minus(1);

// Approve Chai contract to spend maker's Dai
await daiToken
  .approve(chaiToken.address, MAX_UINT256)
  .awaitTransactionSuccess({ from: makerAddress });

// 10 Dai
const daiAmount = Web3Wrapper.toBaseUnitAmount(10, 18);

// Convert maker's Dai to Chai
await chaiToken
  .join(makerAddress, daiAmount)
  .awaitTransactionSuccess({ from: makerAddress });

// Approve ChaiBridge contract to spend maker's Chai
await chaiToken.approve(chaiBridge.address, MAX_UINT256);
```

## Creating a ChaiBridge order

The [`makerAssetData`](../v3/v3-specification.md#orders) of an order must be encoded using [`ERC20Bridge` assetData](../asset-proxy/erc20-bridge-proxy.md#encoding-assetdata) which specifies the `ChaiBridge` contract:

```solidity
bytes memory makerAssetData = abi.encodeWithSelector(
    // Id of ERC20BridgeProxy
    0xdc1600f3,
    // Dai mainnet address. This field will be ignored by the `ChaiBridge` contract. However,
    // it should be specified in order to standardize the parsing of `assetData` across different bridges.
    0x6B175474E89094C44Da98b954EedeAC495271d0F,
    // ChaiBridge mainnet address
    0x77C31EbA23043B9a72d13470F3A3a311344D7438,
    // The `bridgeData` field will be ignored.
    ""
);
```

This results in the following `makerAssetData`: `0xdc1600f30000000000000000000000006b175474e89094c44da98b954eedeac495271d0f00000000000000000000000077c31eba23043b9a72d13470f3a3a311344d743800000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000000`.

## Filling a ChaiBridge order

Once a maker has completed the prerequisite setup and created an order with valid `ChaiBridge` `makerAssetData`, a taker can fill the order via the [Exchange contract](https://github.com/0xProject/0x-protocol-specification/blob/master/v3/v3-specification.md#exchange) with no additonal allowances. These orders are functionally equivalent to a normal Dai order, where the `makerAssetAmount` of the order specifies the amount of Dai that will be transferred. The only difference to the taker is slightly increased gas costs (currently approximately 150K extra gas).
