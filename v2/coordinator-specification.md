1.  [Architecture](#architecture)
1.  [Contracts](#contracts)
    1.  [Coordinator](#coordinator)
    1.  [CoordinatorRegistry](#coordinator)
1.  [Standard Coordinator API](#standard-coordinator-api)

# Architecture

// TODO

- Overview
- Step-by-step of filling a Coordinator order
- Contract logic
- SCA

# Contracts

// TODO

# Standard Coordinator API

In order to ensure that trading clients know how to successfully request a signature from your Coordinator server, it must strictly adhere to the following API specification. If it does not, you risk traders being unable to fill your orders.

## Errors

// TODO

## Rest API

### POST /v1/request_transaction

Submit a signed 0x transaction encoding either a 0x fill or cancellation. If the 0x transaction encodes a fill, the sender is requesting a coordinator signature required to fill the order on-chain. If the 0x transaction encodes an order cancellation request, the sender is requesting the included order(s) to be soft-cancelled by the coordinator.

#### Payload

// TODO: Add payload schema here
[See payload schema](TODO)

```
{
    "signedTransaction": {
        "salt": "62799490869714002130337144204631467715674661702713158022807378407578488540214",
        "signerAddress": "0xe834ec434daba538cd1b9fe1582052b880bd7e63",
        "data": "0xb4be83d500000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000000002a0000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f84000000000000000000000000000000000000000000000000000000000000000000000000000000000000000078dc5d2d739606d31509c31d654056a45185ecb60000000000000000000000006ecbe1db9ef729cbe972c83fb886247691fb6beb0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000ad78ebc5ac62000000000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000005c879733d9cbe51a7e6b6c5e1f057d3d3ba3c1b57c8734fe0f523afa6250aff64aa9db37000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001e00000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e30000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000421c9e99c16367e608d75d943549b49fbeedcf58c84897b69daf0646e6184086a44107dda6e675826189c4bf509061880b5fe3e4707f449203249b98a39480bbee4803000000000000000000000000000000000000000000000000000000000000",
        "verifyingContractAddress": "0x48bacb9266a570d521063ef5dd96e61686dbe788",
        "signature": "0x1b36fca95c03d4e28b0ffb7796194841612005f0913e60f9da36f477a432e478240af81e8be55368b1a43f21266c6964e2b3d6c55b9d11dd5318ab76e63ff0c6f903"
    },
    "txOrigin": "0xe834ec434daba538cd1b9fe1582052b880bd7e63"
}
```

- `signedTransaction` - A signed [0x transaction](https://github.com/0xProject/0x-protocol-specification/blob/ad13141d9a2c6d93e06658d18c53e9f3d99442d4/v2/v2-specification.md#transactions) along with the `verifyingContractAddress` which is part of it's EIP712 signature [domain](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md#definition-of-domainseparator).
- `txOrigin` - The address that will eventually submit the Ethereum transaction executing this 0x transaction on-chain. This will be enforced by the Coordinator extension contract.

#### Response

**Fill request response:**

// TODO: Add response schema
[See fill response schema](TODO)

```
{
    "signature": "0x1cc07d7ae39679690a91418d46491520f058e4fb14debdf2e98f2376b3970de8512ace44af0be6d1c65617f7aae8c2364ff63f241515ee1559c3eeecb0f671d9e903",
    "expirationTimeSeconds": 1552390014
}
```

- `signature` - the coordinator signature required to fill the order
- `expirationTimeSeconds` - when the signature will expire and no longer be valid

**Cancellation request response:**

// TODO: Add response schema
[See cancellation response schema](TODO)

```
{
    "outstandingSignatures": [
        {
            "orderHash": "0xd1dc61f3e7e5f41d72beae7863487beea108971de678ca00d903756f842ef3ce",
            "coordinatorSignature": "0x1c7383ca8ebd6de8b5b20b1c2d49bea166df7dfe4af1932c9c52ec07334e859cf2176901da35f4480ceb3ab63d8d0339d851c31929c40d88752689b9a8a535671303",
            "expirationTimeSeconds": 1552390380,
            "takerAssetFillAmount": "100000000000000000000"
        }
    ]
}
```

- `outstandingSignatures` - Information about the outstanding, unexpired signatures to fill the order(s) that have been soft-cancelled. Once they expire, the maker will have 100% certainty that their order can no longer be filled.
