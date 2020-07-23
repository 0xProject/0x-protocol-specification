# Exchange Proxy Feature: `SignatureValidator`

## Summary

A feature for validating signatures inn Exchange Proxy.

## Motivation

Many features will likely require some form of signature validation (e.g., [MetaTransactions](./meta-transactions.md). Rather than having each feature ship its own validation logic it makes sense to encapsulate this logic into a dedicated and upgradeable feature that they can all share. This way, we can support new signature types and patch vulnerabilities all at once.

## Architecture

The `SignatureValidator` is entirely stateless and exposes two functions for validating hash-based signatures:
- `validateHashSignature(bytes32 hash, address signer, bytes signature)`
  - Validates that `hash` was signed by `signer`, given `signature`. Reverts otherwise.
- `isValidHashSignature(bytes32 hash, address signer, bytes signature)`
  - Returns true if `hash` was signed by `signer`, given `signature`.

## Implementation

### Format
Signatures are constructed similar to those used by the [V3 Exchange](../../v3/v3-specification.md#signature-types):

```solidity
abi.encodePacked(bytes(signature), uint8(signatureType))
```

Where the contents of `signature` depends on the signature type.

### Types
Initially this feature will only support the basic offline signature types, i.e., EIP712 and `eth_sign`, against a hash. The `Illegal` and `Invalid` failing signature types from the V3 Exchange contract will also be handled to simplify migration.

```solidity
/// @dev Allowed signature types.
enum SignatureType {
    Illegal,                     // 0x00, default value
    Invalid,                     // 0x01
    EIP712,                      // 0x02
    EthSign,                     // 0x03
    NSignatureTypes              // 0x04, number of signature types. Always leave at end.
}
```

- **Illegal**
  - This signature type always fails, regardless of signature data.
- **Invalid**
  - Also always failing, regardless of signature data.
- **EIP712**
  - A signature generated via [EIP712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md)'s `eth_signTypedData` RPC call *or* by directly signing a hash.
- **EthSign**
  - A signature generated via the `eth_sign` RPC call (see [JSON-RPC reference](https://eth.wiki/json-rpc/API)).
