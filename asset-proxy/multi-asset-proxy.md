# MultiAssetProxy

## ZEIP-23

The `MultiAssetProxy` was first proposed in [ZEIP-23](https://github.com/0xProject/ZEIPs/issues/23). Please refer to the ZEIP for information and discussion about how this contract works at a higher level.

## Transferring multiple assets

The `MultiAssetProxy` can transfer arbitrary bundles of assets in a single smart contract call. It expects a `values` (`uint256` array) and a `nestedAssetData` (array of [`assetData`](../v3/v3-specification.md#assetdata) byte arrays) to be encoded within its own `assetData`. Each element of `values` corresponds to an element at the same index of `nestedAssetData`. The `MultiAssetProxy` will multiply each `values` element by the `amount` passed into `MultiAssetProxy.transferFrom` and then dispatch the corresponding element of `nestedAssetProxy` to the relevant [`AssetProxy`](../v3/v3-specification.md#assetproxy) contract with the resulting `totalAmount`. This contract does not perform any `transferFrom` calls to assets directly and therefore does not require any additional user approvals.

### transferFrom

This contract may dispatch transfers to other [`AssetProxy`](../v3/v3-specification.md#assetproxy) contracts if its `transferFrom` method is called from an authorized address.

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

Calling `MultiAssetProxy.transferFrom` will perform the following steps:

1. Decode `values` and `nestedAssetData` from `assetData`
1. For each element of `values`, given index `i`:
   1. Multiply `values[i]` by `amount`, resulting in a `scaledValue`
   1. Decode an `assetProxyId` from `nestedAssetData[i]`
   1. Load an `assetProxyAddress` that corresponds to the `assetProxyId`
   1. Call `AssetProxy(assetProxyAddress).transferFrom(nestedAssetData[i], from, to, scaledValue)`
   1. Revert if the call was unsuccessful

#### Errors

The `transferFrom` method may revert with the following errors:

| Error                                                                                       | Condition                                                                                                 |
| ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| [StandardError("SENDER_NOT_AUTHORIZED")](../v3/v3-specification.md#standard-error)          | `msg.sender` has not been authorized                                                                      |
| [StandardError("INVALID_ASSET_DATA_LENGTH")](../v3/v3-specification.md#standard-error)      | The `assetData` is shorter than 68 bytes or is not a multiple of 32 (exluding the 4 byte id)              |
| [StandardError("INVALID_ASSET_DATA_END")](../v3/v3-specification.md#standard-error)         | The offset to `assetData` points to outside the end of calldata                                           |
| [StandardError("LENGTH_MISMATCH")](../v3/v3-specification.md#standard-error)                | The lengths of `values` and `nestedAssetData` are not equal                                               |
| [StandardError("UINT256_OVERFLOW)](../v3/v3-specification.md#standard-error)                | The multiplication of an element of `values` and `amount` resulted in an overflow                         |
| [StandardError("LENGTH_GREATER_THAN_3_REQUIRED")](../v3/v3-specification.md#standard-error) | An element of `nestedAssetData` is shorter than 4 bytes                                                   |
| [StandardError("ASSET_PROXY_DOES_NOT_EXIST")](../v3/v3-specification.md#standard-error)     | No `AssetProxy` contract exists for the given `assetProxyId` of an element of `nestedAssetData`           |
| \*                                                                                          | This contract will rethrow any revert data received from an unsuccessful call of an `AssetProxy` contract |

## Encoding assetData

This contract expects MultiAsset [`assetData`](../v3/v3-specification.md#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following function signature. The id of this contract is `0x94cfcdd7`, which can be calculated as the [4 byte function selector](https://solidity.readthedocs.io/en/latest/abi-spec.html#function-selector) of the same signature.

```solidity
/// @dev Function signature for encoding MultiAsset assetData.
/// @param values Array of values that correspond to each asset to be transferred.
///        Note that each value will be multiplied by the amount being filled in the order before transferring.
/// @param nestedAssetData Array of assetData fields that will be be dispatched to their correspnding AssetProxy contract.
function MultiAsset(
    uint256[] calldata values,
    bytes[] calldata nestedAssetData
)
    external;
```

In Solidity, this data can be encoded with:

```solidity
bytes memory data = abi.encodeWithSelector(
    0x94cfcdd7,
    values,
    nestedAssetData
);
```

Each element of `nestedAssetData` must be encoded according to the specification of the corresponding `AssetProxy` contract. Note that initially, the `MultiAssetProxy` will not support dispatching a transfer to itself.

## Authorizations

The `MultiAssetProxy` has the following interface for managing which addresses are allowed to call this contract's `transferFrom` method. These authorization functions can only be called by the contract's `owner` (currently, the [`AssetProxyOwner`](../v3/v3-specification.md#assetproxyowner) contract).

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

The contracts that are currently authorized to call the `MultiAssetProxy` contract's `transferFrom` method are:

- [Exchange 2.0](../v2/v2-specification.md#exchange)
- [Exchange 3.0](../v3/v3-specification.md#exchange)

## Registering AssetProxy contracts

The `MultiAssetProxy` can only dispatch transfers to other `AssetProxy` contracts that are registered within this contract. The `MultiAssetProxy` has the following interface for managing which `AssetProxy` contracts it is allowed to call. New registrations can only be initiated by the `owner` of this contract (currently, the [`AssetProxyOwner`](../v3/v3-specification.md#assetproxyowner) contract).

```solidity
contract IAssetProxyDispatcher {

    // Logs registration of new asset proxy
    event AssetProxyRegistered(
        bytes4 id,              // Id of new registered AssetProxy.
        address assetProxy      // Address of new registered AssetProxy.
    );

    /// @dev Registers an asset proxy to its asset proxy id.
    ///      Once an asset proxy is registered, it cannot be unregistered.
    /// @param assetProxy Address of new asset proxy to register.
    function registerAssetProxy(address assetProxy)
        external;

    /// @dev Gets an asset proxy.
    /// @param assetProxyId Id of the asset proxy.
    /// @return The asset proxy registered to assetProxyId. Returns 0x0 if no proxy is registered.
    function getAssetProxy(bytes4 assetProxyId)
        external
        view
        returns (address);
}

```

The `AssetProxy` contracts that are currently registered withing the `MultiAssetProxy` are:

- [`ERC20Proxy`](../asset-proxy/erc20-proxy.md)
- [`ERC721Proxy`](../asset-proxy/erc721-proxy.md)
- [`ERC1155Proxy`](../asset-proxy/erc1155-proxy.md)
- [`StaticCallProxy`](../asset-proxy/static-call-proxy.md)
