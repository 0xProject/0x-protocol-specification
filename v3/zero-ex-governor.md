# ZeroExGovernor

The `ZeroExGovernor` is a time-locked multi-signature wallet that has permission to perform administrative functions within the protocol. By default, submitted transactions must pass a 14 day timelock before they are executed. However, the `ZeroExGovernor` also allows custom timelocks to be registered to arbitrary function calls at different destination addresses. Many functions that can be used to mitigate damage in case of emergencies (for example, if a vulnerability is discovered that puts user funds at risk) do not have a timelock. Other functions that affect the system of staking contracts have timelocks that are denominated in terms of 10 day epochs in order to maximize security of upgrades.

The `ZeroExGovernor` has authorizations to perform the following functions within the protocol:

## Managing ownership of all contracts

The `ZeroExGovernor` can transfer ownership of any contract for which it is the `owner` by calling the following function:

```solidity
/// @dev Transfers ownership to a new address.
/// @param newOwner Address of the new owner.
function transferOwnership(address newOwner)
    public;
```

## Managing authorizations in the StakingProxy, ZrxVault, and AssetProxy contracts

Most [`AssetProxy`](./v3-specification.md#assetproxy) require that the caller of their `transferFrom` function is authorized to make the call. The `ZeroExGovernor` is responsible for adding or removing authorizations (and may bypass the timelock when removing an authorization). This is also the mechanism used for upgrading the `Exchange` contract without redeploying each individual `AssetProxy`. A new `Exchange` contract can be authorized while the authorizations of old `Exchange` contracts are removed. Multiple contracts can also be simultaneously authorized.

The `ZeroExGovernor` can also manage authorizations for the `StakingProxy` and `ZrxVault` contracts. While the `ZeroExGovernor` itself is currently the only authorized address in these contracts, this feature can be used to allow new contracts to perform admin functions under different conditions in the future (such as with an on-chain token vote).

## Registering AssetProxy contracts in the Exchange and MultiAssetProxy

[`AssetProxy`](./v3-specification.md#assetproxy) contracts must be registered in the `Exchange` and `MultiAssetProxy` contracts in order to be utilized when filling orders. The `ZeroExGovernor` can register new `AssetProxy` contracts by calling the following function:

```solidity
/// @dev Registers an asset proxy to its asset proxy id.
///      Once an asset proxy is registered, it cannot be unregistered.
/// @param assetProxy Address of new asset proxy to register.
function registerAssetProxy(address assetProxy)
    external;
```

## Setting the protocol fee multiplier in the Exchange

The `ZeroExGovernor` can update the [protocol fee multiplier](./v3-specification.md#calculating-the-protocol-fee) by calling the following function:

```solidity
/// @dev Allows the owner to update the protocol fee multiplier.
/// @param updatedProtocolFeeMultiplier The updated protocol fee multiplier.
function setProtocolFeeMultiplier(uint256 updatedProtocolFeeMultiplier)
    external;
```

Setting the `protocolFeeMultiplier` will emit a [`ProtocolFeeMultiplier`](./v3-specification.md#protocolfeemultiplier) event.

## Setting the protocol fee collector (StakingProxy) in the Exchange

The `ZeroExGovernor` can update the protocol fee collector contract (currently the [Staking](./v3-specification.md#staking) contract) by calling either of the following functions:

```solidity
/// @dev Allows the owner to update the protocolFeeCollector address.
/// @param updatedProtocolFeeCollector The updated protocolFeeCollector contract address.
function setProtocolFeeCollectorAddress(address updatedProtocolFeeCollector)
    external;

/// @dev Sets the protocolFeeCollector contract address to 0.
///      Only callable by owner.
function detachProtocolFeeCollector()
    external;
```

Setting the `protocolFeeCollector` will emit a [`ProtocolFeeCollectorAddress`](./v3-specification.md#protocolfeecollectoraddress) event.

## Updating staking parameters in the StakingProxy

The `ZeroExGovernor` can update individual staking parameters by calling the following function:

```solidity
/// @dev Set all configurable parameters at once.
/// @param _epochDurationInSeconds Minimum seconds between epochs.
/// @param _rewardDelegatedStakeWeight How much delegated stake is weighted vs operator stake, in ppm.
/// @param _minimumPoolStake Minimum amount of stake required in a pool to collect rewards.
/// @param _cobbDouglasAlphaNumerator Numerator for cobb douglas alpha factor.
/// @param _cobbDouglasAlphaDenominator Denominator for cobb douglas alpha factor.
function setParams(
    uint256 _epochDurationInSeconds,
    uint32 _rewardDelegatedStakeWeight,
    uint256 _minimumPoolStake,
    uint32 _cobbDouglasAlphaNumerator,
    uint32 _cobbDouglasAlphaDenominator
)
    external;
```

## Adding or removing Exchange contracts that are allowed to pay protocol fees to the StakingProxy

The `ZeroExGovernor` can add or remove an `Exchange` contract from the `StakingProxy` by calling either of the following functions:

```solidity
/// @dev Adds a new exchange address
/// @param addr Address of exchange contract to add
function addExchangeAddress(address addr)
    external;

/// @dev Removes an existing exchange address
/// @param addr Address of exchange contract to remove
function removeExchangeAddress(address addr)
    external;
```

## Upgrading the staking logic contract that is attached to the StakingProxy

The `ZeroExGovernor` can upgrade the logic of the StakingProxy by calling either of the following functions:

```solidity
/// @dev Attach a staking contract; future calls will be delegated to the staking contract.
/// Note that this is callable only by an authorized address.
/// @param _stakingContract Address of staking contract.
function attachStakingContract(address _stakingContract)
    external;

/// @dev Detach the current staking contract.
/// Note that this is callable only by an authorized address.
function detachStakingContract()
    external;
```

## Setting the StakingProxy that is allowed to trigger deposits and withdrawals from the ZrxVault

The `ZeroExGovernor` can replace the `StakingProxy` contract that triggers deposits and withdrawals in the `ZrxVault` by calling the following function:

```solidity
/// @dev Sets the address of the StakingProxy contract.
/// Note that only the contract owner can call this function.
/// @param _stakingProxyAddress Address of Staking proxy contract.
function setStakingProxy(address _stakingProxyAddress)
    external;
```

## Setting the AssetProxy that is used used to deposit to the ZrxVault

The `ZeroExGovernor` can replace the `AssetProxy` contract used to perform deposits into the `ZrxVault` by calling the following function:

```solidity
/// @dev Sets the Zrx proxy.
/// Note that only an authorized address can call this function.
/// Note that this can only be called when *not* in Catastrophic Failure mode.
/// @param _zrxProxyAddress Address of the 0x Zrx Proxy.
function setZrxProxy(address _zrxProxyAddress)
    external;
```

## Entering catastrophic failure mode in the ZrxVault

The `ZeroExGovernor` can enter catastrophic failure mode in the `ZrxVault` in emergencies by calling the following function:

```solidity
/// @dev Vault enters into Catastrophic Failure Mode.
/// *** WARNING - ONCE IN CATOSTROPHIC FAILURE MODE, YOU CAN NEVER GO BACK! ***
/// Note that only the contract owner can call this function.
function enterCatastrophicFailure()
    external;
```

## List of all administrative functions and timelocks

Function timelocks are represented in days, where one day is equivalent to 86,400 seconds. Custom timelocks are in bold.

| Contract         | Function                         | Selector | Timelock    |
| ---------------- | -------------------------------- | -------- | ----------- |
| Exchange         | `registerAssetProxy`             | c585bb93 | 14 days     |
| Exchange         | `setProtocolFeeMultiplier`       | 9331c742 | **7 days** |
| Exchange         | `setProtocolFeeCollectorAddress` | c0fa16cc | **14 days** |
| Exchange         | `detachProtocolFeeCollector`     | 0efca185 | **0 days**  |
| Exchange         | `transferOwnership`              | f2fde38b | 14 days     |
| StakingProxy     | `addExchangeAddress`             | 8a2e271a | **14 days** |
| StakingProxy     | `removeExchangeAddress`          | 01e28d84 | **14 days** |
| StakingProxy     | `attachStakingContract`          | 66615d56 | **14 days** |
| StakingProxy     | `detachStakingContract`          | 37b006a6 | **14 days** |
| StakingProxy     | `setParams`                      | 9c3ccc82 | **7 days** |
| StakingProxy     | `addAuthorizedAddress`           | 42f1181e | **14 days** |
| StakingProxy     | `removeAuthorizedAddress`        | 70712939 | **14 days** |
| StakingProxy     | `removeAuthorizedAddressAtIndex` | 9ad26744 | **14 days** |
| StakingProxy     | `transferOwnership`              | f2fde38b | **14 days** |
| ZrxVault         | `setStakingProxy`                | 6bf3f9e5 | **14 days** |
| ZrxVault         | `enterCatastrophicFailure`       | c02e5a7f | **0 days**  |
| ZrxVault         | `setZrxProxy`                    | ca5b0218 | **14 days** |
| ZrxVault         | `addAuthorizedAddress`           | 42f1181e | **14 days** |
| ZrxVault         | `removeAuthorizedAddress`        | 70712939 | **14 days** |
| ZrxVault         | `removeAuthorizedAddressAtIndex` | 9ad26744 | **14 days** |
| ZrxVault         | `transferOwnership`              | f2fde38b | **14 days** |
| ERC20Proxy       | `addAuthorizedAddress`           | 42f1181e | 14 days     |
| ERC20Proxy       | `removeAuthorizedAddress`        | 70712939 | **0 days**  |
| ERC20Proxy       | `removeAuthorizedAddressAtIndex` | 9ad26744 | **0 days**  |
| ERC20Proxy       | `transferOwnership`              | f2fde38b | 14 days     |
| ERC721Proxy      | `addAuthorizedAddress`           | 42f1181e | 14 days     |
| ERC721Proxy      | `removeAuthorizedAddress`        | 70712939 | **0 days**  |
| ERC721Proxy      | `removeAuthorizedAddressAtIndex` | 9ad26744 | **0 days**  |
| ERC721Proxy      | `transferOwnership`              | f2fde38b | 14 days     |
| ERC1155Proxy     | `addAuthorizedAddress`           | 42f1181e | 14 days     |
| ERC1155Proxy     | `removeAuthorizedAddress`        | 70712939 | **0 days**  |
| ERC1155Proxy     | `removeAuthorizedAddressAtIndex` | 9ad26744 | **0 days**  |
| ERC1155Proxy     | `transferOwnership`              | f2fde38b | 14 days     |
| ERC20BridgeProxy | `addAuthorizedAddress`           | 42f1181e | 14 days     |
| ERC20BridgeProxy | `removeAuthorizedAddress`        | 70712939 | **0 days**  |
| ERC20BridgeProxy | `removeAuthorizedAddressAtIndex` | 9ad26744 | **0 days**  |
| ERC20BridgeProxy | `transferOwnership`              | f2fde38b | 14 days     |
| MultiAssetProxy  | `addAuthorizedAddress`           | 42f1181e | 14 days     |
| MultiAssetProxy  | `removeAuthorizedAddress`        | 70712939 | **0 days**  |
| MultiAssetProxy  | `removeAuthorizedAddressAtIndex` | 9ad26744 | **0 days**  |
| MultiAssetProxy  | `transferOwnership`              | f2fde38b | 14 days     |
| MultiAssetProxy  | `registerAssetProxy`             | c585bb93 | 14 days     |
