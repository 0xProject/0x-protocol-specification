# ERC1155Proxy

## ZEIP-24

The `ERC1155Proxy` was first proposed in [ZEIP-24](https://github.com/0xProject/ZEIPs/issues/24). Please refer to the ZEIP for information and discussion about how this contract works at a higher level.

## Transferring ERC-1155 tokens

The `ERC1155Proxy` is responsible for transferring [ERC-1155 tokens](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1155.md). Users must first approve this contract by calling the `setApprovalForAll` method on the token that will be exchanged.

The `ERC1155Proxy` utilizes an ERC-1155 token's `safeBatchTransferFrom` method to transfer multiple tokens at once, if desired. Values of each `tokenId` to be transferred are scaled similarly to the method used in the [`MultiAssetProxy`](../asset-proxy/multi-asset-proxy.md). `values` (`uint256` array) and `tokenIds` (`uint256` array) parameters are encoded within the `assetData` that can be utilized by this contract's `transferFrom` method. Each element of `values` corresponds to an element at the same index of `tokenIds`. The `ERC1155Proxy` will multiply each `values` element by the `amount` passed into `ERC1155Proxy.transferFrom` and then pass the resulting scaled values into `ERC1155Token.safeBathTransferFrom` when performing the transfer.

### transferFrom

This contract may transfer ERC-1155 tokens if its `transferFrom` method is called from an authorized address.

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

Calling `ERC1155Proxy.transferFrom` will perform the following steps:

1. Decode the `erc1155TokenAddress`, `tokenIds`, `values`, and `callbackData` from `assetData`
1. Multiply each element of `values` by `amount`, resulting in a `scaledValues` array
1. Call `ERC1155Token(erc1155TokenAddress).safeBatchTransferFrom(from, to, tokenIds, scaledValues, callbackData)`
1. Revert if the call was unsuccessful

#### Errors

The `transferFrom` method may revert with the following errors:

| Error                                                                                   | Condition                                                                                                             |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| [StandardError("SENDER_NOT_AUTHORIZED")](../v3/v3-specification.md#standard-error)      | `msg.sender` has not been authorized                                                                                  |
| [StandardError("FROM_LESS_THAN_TO_REQUIRED")](../v3/v3-specification.md#standard-error) | `assetData` is shorter than 4 bytes                                                                                   |
| [StandardError("UINT256_OVERFLOW")](../v3/v3-specification.md#standard-error)           | The multiplication of an element of `assetData.values` and `amount` resulted in an overflow                           |
| \*                                                                                      | This contract will rethrow any revert data received from an unsuccessful call of `ERC1155Token.safeBatchTransferFrom` |

## Encoding assetData

This contract expects ERC-1155 [`assetData`](../v3/v3-specification.md#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following function signature. The id of this contract is `0xa7cb5fb7`, which can be calculated as the [4 byte function selector](https://solidity.readthedocs.io/en/latest/abi-spec.html#function-selector) of the same signature.

```solidity
/// @dev Function signature for encoding ERC-1155 assetData.
/// @param tokenAddress Address of ERC-1155 token contract.
/// @param tokenIds Array of ids of tokens to be transferred.
/// @param values Array of values that correspond to each token id to be transferred.
///        Note that each value will be multiplied by the amount being filled in the order before transferring.
/// @param callbackData Extra data to be passed to receiver's `onERC1155Received` callback function.
function ERC1155Assets(
    address tokenAddress,
    uint256[] calldata tokenIds,
    uint256[] calldata values,
    bytes calldata callbackData
)
    external;
```

In Solidity, this data can be encoded with:

```solidity
bytes memory data = abi.encodeWithSelector(
    0xa7cb5fb7,
    erc1155TokenAddress,
    tokenIds,
    values,
    callbackData
);
```

NOTE: The `ERC1155Proxy` does not enforce strict length checks for `assetData`, which means that extra data may be appended to this field with any arbitrary encoding. Any extra data will be ignored by the `ERC1155Proxy` but may be used in external contracts interacting with the [`Exchange`](../v3/v3-specification.md#exchange) contract. Relayers that do not desire this behavior should validate the length of all `assetData` fields contained in [orders](../v3/v3-specification.md#orders) before acceptance.

## Authorizations

The `ERC1155Proxy` has the following interface for managing which addresses are allowed to call this contract's `transferFrom` method. These authorization functions can only be called by the contract's `owner` (currently, the [`AssetProxyOwner`](v3/v3-specification.md#assetproxyowner) contract).

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

The contracts that are currently authorized to call the `ERC20Proxy` contract's `transferFrom` method are:

- [Exchange 2.0](../v2/v2-specification.md#exchange)
- [Exchange 3.0](../v3/v3-specification.md#exchange)
- [MultiAssetProxy](../asset-proxy/multi-asset-proxy.md)
