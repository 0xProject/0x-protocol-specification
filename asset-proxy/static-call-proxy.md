# StaticCallProxy

## ZEIP-39

The `StaticCallProxy` was first proposed in [ZEIP-39](https://github.com/0xProject/ZEIPs/issues/39). Please refer to the ZEIP for information and discussion about how this contract works at a higher level.

## Conditional transfers

The `StaticCallProxy` itself cannot perform any transfers. Instead, it is primarily intended to be used in conjunction with the [`MultiAssetProxy`](../asset-proxy/multi-asset-proxy.md) to validate some on-chain state before or after any other transfers are performed. As implied in its name, the `StaticCallProxy` uses the `STATICCALL` opcode described in [EIP-214](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md) to ensure that it cannot modify any on-chain state. Therefore, this contract does not require any user approvals to be used.

### transferFrom

This contract may perform a `STATICCALL` and verify its result if its `transferFrom` method is called.

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

Calling `StaticCallProxy.transferFrom` will perform the following steps:

1. Decode `staticCallTargetAddress`, `staticCallData`, and `expectedReturnDataHash` from `assetData`
1. Call `staticCallTargetAddress.staticcall(staticCallData)`
1. Revert if the call was unsuccessful
1. Calculate the Keccak-256 hash of the return data, resulting in `returnDataHash`
1. Revert if the `returnDataHash` does not exactly match the `expectedReturnDataHash`

#### Errors

The `transferFrom` method may revert with the following errors:

| Error                                                                                      | Condition                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| [StandardError("UNEXPECTED_STATIC_CALL_RESULT")](../v3/v3-specification.md#standard-error) | The hash of the return data does not match the `expectedReturnDataHash`                                                               |
| \*                                                                                         | This contract will rethrow any revert data received from an unsuccessful call of `staticCallTargetAddress.staticcall(staticCallData)` |

## Encoding assetData

This contract expects StaticCall [`assetData`](../v3/v3-specification.md#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following function signature. The id of this contract is `0xc339d10a`, which can be calculated as the [4 byte function selector](https://solidity.readthedocs.io/en/latest/abi-spec.html#function-selector) of the same signature.

```solidity
/// @dev Function signature for encoding StaticCall assetData.
/// @param staticCallTargetAddress Target address of staticcall.
/// @param staticCallData Data that will be passed to staticCallTargetAddress in the staticcall.
/// @param expectedReturnDataHash Expected Keccak-256 hash of the staticcall return data.
function StaticCall(
    address staticCallTargetAddress,
    bytes calldata staticCallData,
    bytes32 expectedReturnDataHash
)
    external;
```

In Solidity, this data can be encoded with:

```solidity
bytes memory data = abi.encodeWithSelector(
    0xc339d10a,
    staticCallTargetAddress,
    staticCallData,
    expectedReturnDataHash
);
```

## Authorizations

Since this contract can only read from state, no authorizations are required. This contract's `transferFrom` method can be called by any address.
