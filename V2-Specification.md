TODO: Update transactions with EIP712
TODO: architecture section with diagram
TODO: Sections on AssetProxy contracts, AssetProxyOwner, and AssetProxyDispatcher
TODO: Make sure all links work
TODO: Update table of contents
TODO: Errors

1.  [Architecture](#architecture)
1.  [Orders](#orders)
1.  [Transactions](#transactions)
1.  [Signatures](#signatures)
1.  [Contracts](#contracts)
1.  [Recommendations](#recommendations)

# Architecture

# Orders

## Message format

An order message consists of the following parameters:

| Parameter                    | Type    | Description                                                                                                                                      |
| ---------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| makerAddress                 | address | Address that created the order.                                                                                                                  |
| takerAddress                 | address | Address that is allowed to fill the order. If set to 0, any address is allowed to fill the order.                                                |
| feeRecipientAddress          | address | Address that will recieve fees when order is filled.                                                                                             |
| [senderAddress](#sender)     | address | Address that is allowed to call Exchange contract methods that take orders as inputs. If set to 0, any address is allowed to call these methods. |
| makerAssetAmount             | uint256 | Amount of makerAsset being offered by maker. Must be greater than 0.                                                                             |
| takerAssetAmount             | uint256 | Amount of takerAsset being bid on by maker. Must be greater than 0.                                                                              |
| makerFee                     | uint256 | Amount of ZRX paid to feeRecipient by maker when order is filled. If set to 0, no transfer of ZRX from maker to feeRecipient will be attempted.  |
| takerFee                     | uint256 | Amount of ZRX paid to feeRecipient by taker when order is filled. If set to 0, no transfer of ZRX from taker to feeRecipient will be attempted.  |
| expirationTimeSeconds        | uint256 | Timestamp in seconds at which order expires.                                                                                                     |
| [salt]()                     | uint256 | Arbitrary number to facilitate uniqueness of the order's hash.                                                                                   |
| [makerAssetData](#salt)      | bytes   | Encoded data that can be decoded by a specified proxy contract when transferring makerAsset. The last byte must reference the id of this proxy.  |
| [takerAssetData](#assetdata) | bytes   | Encoded data that can be decoded by a specified proxy contract when transferring takerAsset. The last byte must reference the id of this proxy.  |

### Sender

### Salt

An order's `salt` parameter has two main usecases:

*   To ensure uniqueness within an order's hash.
*   To be used in combination with [`cancelOrdersUpTo`](#cancelordersupto). To get the most benefit of this usecase, it is recommended that the `salt` field be treated as a timestamp for when orders have been created. A timestamp in milliseconds would allow a `maker` to create 1000 orders with the same parameters per second.

### AssetData

## Hashing an order

The hash of an order is used as a unique identifier of that order. An order is hashed according to the [EIP712 specification](https://github.com/ethereum/EIPs/pull/712/files).

```
bytes32 constant DOMAIN_SEPARATOR_SCHEMA_HASH = keccak256(
    "DomainSeparator(address contract)"
);

bytes32 constant ORDER_SCHEMA_HASH = keccak256(
    "Order(",
    "address makerAddress,",
    "address takerAddress,",
    "address feeRecipientAddress,",
    "address senderAddress,",
    "uint256 makerAssetAmount,",
    "uint256 takerAssetAmount,",
    "uint256 makerFee,",
    "uint256 takerFee,",
    "uint256 expirationTimeSeconds,",
    "uint256 salt,",
    "bytes makerAssetData,",
    "bytes takerAssetData,",
    ")"
);

bytes32 orderHash = keccak256(
    DOMAIN_SEPARATOR_SCHEMA_HASH,
    keccak256(address(this)),
    ORDER_SCHEMA_HASH,
    keccak256(
        order.makerAddress,
        order.takerAddress,
        order.feeRecipientAddress,
        order.senderAddress,
        order.makerAssetAmount,
        order.takerAssetAmount,
        order.makerFee,
        order.takerFee,
        order.expirationTimeSeconds,
        order.salt,
        keccak256(order.makerAssetData),
        keccak256(order.takerAssetData)
    )
);
```

## Creating an order

An order may only be filled if it can be paired with an associated valid signature. Signatures are only validated the first time an order is filled. For later fills, no signature must be submitted. An order's hash must be signed with a [supported signature type](#signature-types).

## Filling orders

Orders can be filled by calling the following methods on the Exchange contract.

### fillOrder

This is the most basic way to fill an order. All of the other methods call `fillOrder` under the hood with additional logic. This function will attempt to fill the amount specified by the caller. However, if the remaining fillable amount is less than the amount specified, the remaining amount will be filled. Partial fills are allowed when filling orders.

`fillOrder` will revert under the following conditions:

*   The caller of `fillOrder` is different from the `sender` specified in the order (unless `sender == address(0)`).
*   The taker of `fillOrder` is different from the `taker` specified in the order (unless `taker == address(0)`).
*   An invalid signature is submitted (this is only checked the first time an order is filled).
*   The `makerAssetAmount` or `takerAssetAmount` specified in the order are equal to 0.
*   The amount that the taker is attempting to fill is 0.
*   The order has expired.
*   The order has been cancelled.
*   The order has already been fully filled.
*   Filling the order results in a rounding error > 0.1% of the `takerAssetAmount` that would otherwise be filled.
*   Any transfers associated with the fill fail.

If successful, `fillOrder` will emit a [`Fill`](#fill) event. If the transaction does not revert, a [`FillResults`](#fillresults) instance will be returned.

```
/// @dev Fills the input order.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
/// @return Amounts filled and fees paid by maker and taker.
function fillOrder(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### fillOrKillOrder

`fillOrKillOrder` behaves almost exactly the same as `fillOrder`. However, the transaction will revert if the amount specified is not filled exactly.

```
/// @dev Fills the input order. Reverts if exact takerAssetFillAmount not filled.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
function fillOrKillOrder(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### fillOrderNoThrow

`fillOrderNoThrow` also behaves very similary to `fillOrder`. However, the transaction will never revert and will instead return a [`FillResults`](#fillresults) instance that contains all 0 values. This is useful when calling the batch methods listed below, where a user may not want an entire transaction to fail when a single fill is reverted.

```
/// @dev Fills an order with specified parameters and ECDSA signature.
///      Returns false if the transaction would otherwise revert.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
/// @return Amounts filled and fees paid by maker and taker.
function fillOrderNoThrow(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### batchFillOrders

`batchFillOrders` calls `fillOrder` sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calls of fillOrder.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
function batchFillOrders(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
```

### batchFillOrKillOrders

`batchFillOrKillOrders` calls `fillOrKillOrder` sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calls of fillOrKill.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
function batchFillOrKillOrders(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
```

### batchFillOrdersNoThrow

`batchFillOrdersNoThrow` calls `fillOrderNoThrow` sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calles of fillOrderNoThrow.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
function batchFillOrdersNoThrow(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
```

### marketSellOrders

`marketSellOrders` calls `fillOrder` sequentially for each provided order and signature until the total `takerAssetAmount` has been sold by `taker`. If successful, `marketSellOrders` returns a [`FillResults`]() instance containing the cumulative amounts filled and fees paid.

Note that `marketSellOrders` assumes that the `takerAssetData` is equal for each order. For any order passed in after the first, the `takerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `takerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of takerAsset is sold by taker.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signatures Proofs that orders have been created by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketSellOrders(
    Order[] memory orders,
    uint256 takerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketSellOrdersNoThrow

`marketSellOrdersNoThrow` calls `fillOrderNoThrow` sequentially for each provided order and signature until the total `takerAssetAmount` has been sold by `taker`. If successful, `marketSellOrdersNoThrow` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

Note that `marketSellOrdersNoThrow` assumes that the `takerAssetData` is equal for each order. For any order passed in after the first, the `takerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `takerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of takerAsset is sold by taker.
///      Returns false if the transaction would otherwise revert.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketSellOrdersNoThrow(
    Order[] memory orders,
    uint256 takerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketBuyOrders

`marketBuyOrders` calls `fillOrder` sequentially for each provided order and signature until the total `makerAssetAmount` has been bought by `taker`. If successful, `marketBuyOrders` returns a [`FillResults`]() instance containing the cumulative amounts filled and fees paid.

Note that `marketBuyOrders` assumes that the `makerAssetData` is equal for each order. For any order passed in after the first, the `makerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `makerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of makerAsset is bought by taker.
/// @param orders Array of order specifications.
/// @param makerAssetFillAmount Desired amount of makerAsset to buy.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketBuyOrders(
    Order[] memory orders,
    uint256 makerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketBuyOrdersNoThrow

`marketBuyOrdersNoThrow` calls `fillOrderNoThrow` sequentially for each provided order and signature until the total `makerAssetAmount` has been bought by `taker`. If successful, `marketBuyOrdersNoThrow` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

```
/// @dev Synchronously executes multiple fill orders in a single transaction until total amount is bought by taker.
///      Returns false if the transaction would otherwise revert.
/// @param orders Array of order specifications.
/// @param makerAssetFillAmount Desired amount of makerAsset to buy.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketBuyOrdersNoThrow(
    Order[] memory orders,
    uint256 makerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

## matchOrders

Two orders that represent a bid and an ask for the same token pair may be matched together if the spread between their respective prices is negative. This is satified by the following equation:

```
(leftOrder.makerAssetAmount * rightOrder.makerAssetAmount) >= (leftOrder.takerAssetAmount * rightOrder.takerAssetAmount)
```

The caller of `matchOrders` is considered the `taker` for each order. The `taker` will pay the `takerFee` for each order, but will also recieve the spread between both orders. The spread is always denominated in terms of the left order's `makerAsset`. No balance is required in order to call `matchOrders`, and the `taker` never holds intermediate balances of either asset.

`matchOrders` will revert if either order fails the validation checks for [fillOrder](#fillOrder). Note that `matchOrders` assumes that `rightOrder.makerAssetData == leftOrder.takerAssetData` and `rightOrder.takerAssetData == leftOrder.makerAssetData`, allowing null byte arrays to be passed in for both `assetData` fields of `rightOrder`. If other `assetData` fields were part of the original `rightOrder`, this function will fail when validating the signature of the `rightOrder`.

If successful, `matchOrders` will emit a [Fill](#fill) event for each matched order.

```
/// @dev Match two complementary orders that have a profitable spread.
///      Each order is filled at their respective price point. However, the calculations are
///      carried out as though the orders are both being filled at the right order's price point.
///      The profit made by the left order goes to the taker (who matched the two orders).
/// @param leftOrder First order to match.
/// @param rightOrder Second order to match.
/// @param leftSignature Proof that order was created by the left maker.
/// @param rightSignature Proof that order was created by the right maker.
/// @return matchedFillResults Amounts filled and fees paid by maker and taker of matched orders.
function matchOrders(
    Order memory leftOrder,
    Order memory rightOrder,
    bytes memory leftSignature,
    bytes memory rightSignature
)
    public
    returns (MatchedFillResults memory matchedFillResults);
```

## Cancelling orders

### cancelOrder

`cancelOrder` cancels the specified order. Partial cancels are not allowed.

`cancelOrder` will revert under the following conditions:

*   The `makerAssetAmount` or `takerAssetAmount` specified in the order are equal to 0.
*   The caller of `cancelOrder` is different from the `sender` specified in the order (unless `sender == address(0)`).
*   The `maker` of the order has not authorized the cancel, either by calling `cancelOrder` through an Ethereum transaction or a [0x transaction](#transactions).
*   The order has expired.
*   The order has already been cancelled.

If successful, `cancelOrder` will emit a [`Cancel`](#cancel) event.

```
/// @dev After calling, the order can not be filled anymore.
/// @param order Order struct containing order specifications.
/// @return True if the order state changed to cancelled.
///         False if the transaction was already cancelled or expired.
function cancelOrder(Order memory order)
    public;
```

### cancelOrdersUpTo

`cancelOrdersUpTo` invalidates all orders created by the caller that have a `salt` value that is less than or equal to the specified `newMakerEpoch` and updates the current `makerEpoch`. This function will revert if `newMakerEpoch` is less than or equal to the current `makerEpoch`. If successful, `cancelOrdersUpTo` will emit a [`CancelUpTo`](#cancelupto) event.

```
/// @dev Cancels all orders created by caller that contain a salt value less than or equal to newMakerEpoch.
/// @param newMakerEpoch Orders created with a salt less or equal to this value will be cancelled.
function cancelOrdersUpTo(uint256 newMakerEpoch)
    external;
```

### batchCancelOrders

`batchCancelOrders` calls `cancelOrder` sequentially for each provided order.

```
/// @dev Synchronously executed musltiple calls of cancelOrder.
/// @param orders Array of order specifications.
function batchCancelOrders(Order[] memory orders)
    public;
```

## Querying state of an order

### filled

The Exchange contract contains a mapping that records the nominal amount of an order's `takerAssetAmount` that has already been filled. This mapping is updated each time an order is successfully filled, allowing for partial fills.

```
// Mapping of orderHash => anominal mount of takerToken already bought by maker
mapping (bytes32 => uint256) public filled;
```

### cancelled

The Exchange contract contains a mapping that records if an order has been cancelled.

```
// Mapping of orderHash => cancelled
mapping (bytes32 => bool) public cancelled;
```

### makerEpoch

The Exchange contract contains a mapping that specifies the `makerEpoch` for each address, which invalidates all orders created by that address that contain a salt value less than or equal to the current `makerEpoch`.

```
// Mapping of makerAddress => lowest salt an order can have in order to be fillable
// Orders with a salt less than their maker's epoch are considered cancelled
mapping (address => uint256) public makerEpoch;
```

### getOrderInfo

`getOrderInfo` is a public method that returns the state, hash, and amount of an order that has already been filled as an [OrderInfo](#orderinfo) instance:

```
/// @dev Gets information about an order: status, hash, and amount filled.
/// @param order Order to gather information on.
/// @return OrderInfo Information about the order and its state.
///         See LibOrder.OrderInfo for a complete description.
function getOrderInfo(Order memory order)
    public
    view
    returns (OrderInfo memory orderInfo);
```

# Transactions

## Message format

| Parameter | Type    | Description                                                                      |
| --------- | ------- | -------------------------------------------------------------------------------- |
| signer    | address | Address of transaction signer                                                    |
| salt      | uint256 | Arbitrary number to facilitate uniqueness of the transactions's hash.            |
| data      | bytes   | The calldata that is to be executed. This must call an Exchange contract method. |

## Hash of a transaction

TODO: EIP712 hash

The hash of a transaction is used as a unique identifier for that transaction. A transaction's hash is defined as the [tightly packed]() Keccak 256 hash of the Exchange contract address and transaction's parameters.

```
bytes32 transactionHash = keccak256(
    address(this),
    signer,
    salt,
    data
);
```

## Creating a transaction

A transaction may only be executed if it can be paired with an associated valid signature. A transaction's hash must be signed with a [supported signature type](#signature-types).

## Executing a transaction

### API

A transaction may only be executed by calling the `executeTransaction` method of the Exchange contract. `executeTransaction` attempts to execute any function on the Exchange contract in the context of the transaction signer (rather than `msg.sender`).

`executeTransaction` will revert under the following conditions:

*   Reentrancy is attempted (e.g `executeTransaction` calls `executeTransaction` again).
*   A transaction with an equivalent hash has already been executed.
*   An invalid signature is submitted.
*   The execution of the provided data reverts. TODO: is this desired behavior?

```
/// @dev Executes an exchange method call in the context of signer.
/// @param salt Arbitrary number to ensure uniqueness of transaction hash.
/// @param signer Address of transaction signer.
/// @param data AbiV2 encoded calldata.
/// @param signature Proof that transaction has been signed by signer.
function executeTransaction(
    uint256 salt,
    address signer,
    bytes data,
    bytes signature
)
    external;
```

### Example

TODO: Link to whitelist contract

# Signatures

## API

The Exchange contract includes a method `isValidSignature` for validating signatures. This method has the following interface:

```
function isValidSignature(
    bytes32 hash,
    address signer,
    bytes memory signature
)
    public view
    returns (bool isValid);
```

## Signature Types

All signatures submitted to the Exchange contract are represented as a byte array of arbitrary length, where the last byte (the "signature byte") specifies the signatures type. The signature type is popped from the signature byte array before validation. The following signature types are supported within the protocol:

| Signature byte | Signature type          |
| -------------- | ----------------------- |
| 0x00           | [Illegal](#illegal)     |
| 0x01           | [Invalid](#invalid)     |
| 0x02           | [EIP712](#eip712)       |
| 0x03           | [EthSign](#ethsign)     |
| 0x04           | [Caller](#caller)       |
| 0x05           | [Wallet](#wallet)       |
| 0x06           | [Validator](#validator) |
| 0x07           | [PreSigned](#presigned) |
| 0x08           | [Trezor](#trezor)       |

### Illegal

The is the default value of the signature byte. A transaction that includes an Illegal signature will be reverted. Therefore, users must explicitly specify a valid signature type.

### Invalid

An Invalid signature always returns false. An invalid signature can always be recreated and is therefore offered explicitly. This signature type is largely used for testing purposes.

### EIP712

An EIP712 signature is considered valid if the address recovered from calling `ecrecover` is the same as the specified signer. In this case, `ecrecover` must be called with the following arguments:

1.  `bytes32 hash`: Any 32 byte hash that adheres to the [EIP712 standard](https://github.com/ethereum/EIPs/pull/712/files).
2.  `uint8 v`: signature[0].
3.  `bytes32 r`: signature[1:32].
4.  `bytes32 s`: signature[33:64].

### EthSign

An EthSign signature is considered valid if the address recovered from calling `ecrecover` is the same as the specified signer. In this case, `ecrecover` must be called with the following arguments:

1.  `bytes32 msgHash`: Created by hashing a 32 hash with the `"\x19Ethereum Signed Message:\n32"` prefix.
2.  `uint8 v`: signature[0].
3.  `bytes32 r`: signature[1:32].
4.  `bytes32 s`: signature[33:64].

### Caller

This signature type will consider the signer to be the sender of the current message call (i.e `msg.sender`).

### Wallet

The Wallet signature type allows a contract to trade on behalf of any other address(es) by defining it's own signature validation function. When used with order signing, the Wallet contract _is_ the `maker` of the order and should hold any assets that will be traded. This contract should have the following interface:

```
contract IWallet {

    /// @dev Verifies that a signature is valid.
    /// @param hash Message hash that is signed.
    /// @param signature Proof of signing.
    /// @return Validity of order signature.
    function isValidSignature(
        bytes32 hash,
        bytes signature
    )
        external
        view
        returns (bool isValid);
}
```

Note when using this method to sign orders: although it can be useful to allow the validity of signatures to be determined by some state stored on the blockchain, it should be noted that the signature will only be checked the first time an order is filled. Therefore, the signature cannot be later invalidated by updating the associates state.

### Validator

The Validator signature type allows an address to delegate signature verification to any other address. The Validator contract must first be approved by calling the following method:

```
// Mapping of signer => validator => approved
mapping (address => mapping (address => bool)) public allowedValidators;

/// @dev Approves a Validator contract to verify signatures on signer's behalf.
/// @param validator Address of Validator contract.
/// @param approval Approval or disapproval of  Validator contract.
function approveSignatureValidator(
    address validator,
    bool approval
)
    external;
```

A Validator signature is then encoded as:

1.  `bytes signature`: signature[0:signature.length - 20 - 1]
2.  `address validator`: signature[signature.length - 20:signature.length]

A Validator contract must have the following interface:

```
contract IValidator {

    /// @dev Verifies that a signature is valid.
    /// @param hash Message hash that is signed.
    /// @param signer Address that should have signed the given hash.
    /// @param signature Proof of signing.
    /// @return Validity of order signature.
    function isValidSignature(
        bytes32 hash,
        address signer,
        bytes signature
    )
        external
        view
        returns (bool isValid);
}
```

The signature is validated by calling the Validator contract's `isValidSignature` method.

```
// Pop last 20 bytes off of signature byte array.
address validator = popAddress(signature);

// Ensure signer has approved validator.
if (!allowedValidators[signer][validator]) {
    return false;
}

isValid = IValidator(validator).isValidSignature(
    hash,
    signer,
    signature
);
```

### PreSigned

Allows any address to sign a hash on-chain by calling the `preSign` method on the Exchange contract.

```
// Mapping of hash => signer => signed
mapping (bytes32 => mapping(address => bool)) public preSigned;

/// @dev Approves a hash on-chain using any valid signature type.
///      After presigning a hash, the preSign signature type will become valid for that hash and signer.
/// @param signer Address that should have signed the given hash.
/// @param signature Proof that the hash has been signed by signer.
function preSign(
    bytes32 hash,
    address signer,
    bytes signature
)
    external;
```

The hash can then be validated with only a PreSigned signature byte by checking the state of the `preSigned` mapping when a transaction is submitted.

```
isValid = preSigned[hash][signer];
return isValid;
```

### Trezor

This signature type allows for compatability with Trezor hardware wallets, which use a non-standard encoding when adding a prefix to the hash being signed. A Trezor signature is considered valid if the address recovered from calling `ecrecover` is the same as the specified signer. In this case, `ecrecover` must be called with the following arguments:

1.  `bytes32 msgHash`: Created by hashing a 32 byte hash with the `"\x19Ethereum Signed Message:\n\x41"` prefix.
2.  `uint8 v`: signature[0].
3.  `bytes32 r`: signature[1:32].
4.  `bytes32 s`: signature[33:64].

# Events

## Fill

A `Fill` event is emmitted when an order is filled.

```
event Fill(
    address indexed makerAddress,
    address takerAddress,
    address indexed feeRecipientAddress,
    uint256 makerAssetFilledAmount,
    uint256 takerAssetFilledAmount,
    uint256 makerFeePaid,
    uint256 takerFeePaid,
    bytes32 indexed orderHash,
    bytes makerAssetData,
    bytes takerAssetData
);
```

## Cancel

A `Cancel` event is emmitted whenever an individual order is cancelled.

```
event Cancel(
    address indexed makerAddress,
    address indexed feeRecipientAddress,
    bytes32 indexed orderHash,
    bytes makerAssetData,
    bytes takerAssetData
);
```

## CancelUpTo

A `CancelUpTo` event is emmitted whenever a `cancelOrdersUpTo` call is successful.

```
event CancelUpTo(
    address indexed makerAddress,
    uint256 makerEpoch
);
```

# Types

## Order

```
struct Order {
    address makerAddress;
    address takerAddress;
    address feeRecipientAddress;
    address senderAddress;
    uint256 makerAssetAmount;
    uint256 takerAssetAmount;
    uint256 makerFee;
    uint256 takerFee;
    uint256 expirationTimeSeconds;
    uint256 salt;
    bytes makerAssetData;
    bytes takerAssetData;
}
```

## FillResults

Fill methods that return a value will return a FillResults instance if successful.

```
struct FillResults {
    uint256 makerAssetFilledAmount;
    uint256 takerAssetFilledAmount;
    uint256 makerFeePaid;
    uint256 takerFeePaid;
}
```

## MatchedFillResults

The `matchOrders` method returns a MatchedFillResults instance if successful.

```
struct MatchedFillResults {
    FillResults left;
    FillResults right;
    uint256 leftMakerAssetSpreadAmount;
}
```

## OrderInfo

The `getOrderInfo` method returns an `OrderInfo` instance.

```
struct OrderInfo {
    uint8 orderStatus;
    bytes32 orderHash;
    uint256 orderTakerAssetFilledAmount;
}
```

# Contracts

## Exchange

## AssetProxy

### ERC20Proxy

### ERC721Proxy

## AssetProxyOwner

# Recommendations

## Optimizing calldata
