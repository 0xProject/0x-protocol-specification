# ERC20BridgeProxy

## ZEIP-39

The `ERC20BridgeProxy` was first proposed in [ZEIP-47](https://github.com/0xProject/ZEIPs/issues/47). Please refer to the ZEIP for information and discussion about how this contract works at a higher level.

## Transfering ERC-20 tokens via a Bride contract

### transferFrom

This contract may transfer an ERC-20 token if its `transferFrom` method is called from an authorized address.

```solidity
/// @dev Transfers assets. Either succeeds or throws.
/// @param assetData Byte array encoded for the respective asset proxy.
/// @param from Address to transfer asset from.
/// @param to Address to transfer asset to.
/// @param amount Amount of asset to transfer.
function transferFrom(
    bytes assetData,
    address from,
    address to,
    uint256 amount
)
    external;
```

#### Logic

Calling `ERC20BridgeProxy.transferFrom` will perform the following steps:

1. Decode `tokenAddress`, `bridgeAddress`, and `bridgeData` from `assetData`
1. Query the `to` account's initial balance of the ERC-20 token at `tokenAddress`
1. Call `IERC20Bridge(bridgeAddress).bridgeTransferFrom(tokenAddress, from, to, amount, bridgeData)`
1. Revert if the call did not return `0xdc1600f3` (this contract's id)
1. Revert if the `to` account's new balance of the ERC-20 token is less than its initial balance plus `amount`

#### Errors

The `transferFrom` method may revert with the following errors:

| Error                                                                        | Condition                                                                                                                                                                     |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [StandardError("BRIDGE_FAILED")](../v3/v3-specification.md#standard-error)   | The `bridgeTransferFrom` call did not return `0xdc1600f3`                                                                                                                     |
| [StandardError("BRIDGE_UNDERPAY")](../v3/v3-specification.md#standard-error) | The `bridgeTransferFrom` call increase the `to` account's balance of the ERC-20 token by at least `amount`                                                                    |
| \*                                                                           | This contract will rethrow any revert data received from an unsuccessful call of `IERC20Bridge(bridgeAddress).bridgeTransferFrom(tokenAddress, from, to, amount, bridgeData)` |

## Encoding assetData

This contract expects ERC20Bridge [`assetData`](../v3/v3-specification.md#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following function signature. The id of this contract is `0xdc1600f3`, which can be calculated as the [4 byte function selector](https://solidity.readthedocs.io/en/latest/abi-spec.html#function-selector) of the same signature.

```solidity
/// @dev Function signature for encoding ERC20Bridge assetData.
/// @param tokenAddress Address of token to transfer.
/// @param bridgeAddress Address of the bridge contract.
/// @param bridgeData Arbitrary data to be passed to the bridge contract.
function ERC20Bridge(
    address tokenAddress,
    address bridgeAddress,
    bytes calldata bridgeData
)
    external;
```

In Solidity, this data can be encoded with:

```solidity
bytes memory data = abi.encodeWithSelector(
    0xdc1600f3,
    tokenAddress,
    bridgeAddress,
    bridgeData
);
```

## Authorizations

The `ERC20BridgeProxy` has the following interface for managing which addresses are allowed to call this contract's `transferFrom` method. These authorization functions can only be called by the contract's `owner` (currently, the [`ZeroExGovernor`](../v3/zero-ex-governor.md) contract).

```solidity
contract IAuthorizable {

    /// @dev Gets all authorized addresses.
    /// @return Array of authorized addresses.
    function getAuthorizedAddresses()
        external
        view
        returns (address[]);

    /// @dev Authorizes an address.
    /// @param target Address to authorize.
    function addAuthorizedAddress(address target)
        external;

    /// @dev Removes authorizion of an address.
    /// @param target Address to remove authorization from.
    function removeAuthorizedAddress(address target)
        external;

    /// @dev Removes authorizion of an address.
    /// @param target Address to remove authorization from.
    /// @param index Index of target in authorities array.
    function removeAuthorizedAddressAtIndex(
        address target,
        uint256 index
    )
        external;
}
```

The contracts that are currently authorized to call the `ERC20BridgeProxy` contract's `transferFrom` method are:

- [Exchange 3.0](../v3/v3-specification.md#exchange)
- [MultiAssetProxy](../asset-proxy/multi-asset-proxy.md)
