# Exchange Proxy Featuer: `TokenSpender`

## Summary
A feature for managing and spending token allowances.

## Motivation

We need a feature that encapsulates the pattern and logic and of moving funds from a user for an operation.

## Architecture

![token-spender](./img/token-spender.png)

### Functions
The `TokenSpender` feature exposes the following functions:
- `getAllowanceTarget()`
  - Returns the address of the allowance target for tokens the Exchange Proxy can spend. *This is different from the Exchange Proxy itself.*
- `getSpendableERC20BalanceOf(IERC20 token, address owner)`
  - Returns the total quantity of an ERC20 token that can be moved from `owner` by the Exchange Proxy, given the current allowance and balance.
- `_spendERC20Tokens(IERC20 token, address owner, address to, uint256 amount)`
  - Move `amount` of `token` from `owner` to `to`. *Only callable from inside the Exchange Proxy itself.*

## Implementation

### AllowanceTarget

Allowances are not set directly on the Exchange Proxy. There are a few good reasons for this:

* Lack of separation of complex logic vs access to funds.
* No easy way to detach allowances in case of a critical vulnerability.
* Allowances do not migrate if we ever redeploy the Exchange proxy.
* Dangling allowances if we decide to migrate the Exchange proxy to use the V3 asset proxies for allowances.

A separate contract, called the `AllowanceTarget`, will be the target for *all* token allowances. This contract is owned by the governor with the Exchange Proxy set as an authorized user.

It exposes a single function, `executeCall(address target, bytes callData)`, which only the Exchange Proxy should be allowed to call. This function will perform a low-level `call()` to `target` passing `callData` as calldata. This allows the Exchange Proxy to perform token transfers from the context of the `AllowanceTarget`.


### `_spendERC20Tokens()`
This function is only callable from inside the Exchange Proxy itself. It moves tokens from an account that has granted the `AllowanceTarget` an allowance by calling `AllowanceTarget.executeCall(token, abi.encode(ERC20.transferFrom.selector, owner, to, amount))`.

## Risks & Mitigations
- Unlike with the V3 Exchange asset proxies, each token standard does not have its own, distinct allowance target, potentially increasing the attack surface with each feature that interacts with the `AllowanceTarget`. Thus, it is essential that the `TokenSpender` feature be the only allowed means of accessing user allowances.
- There is nothing technically preventing another feature from directly access the `AllowanceTarget`. Developers must not allow this pattern to be used.
- The `AllowanceTarget` instance is owned by the governor contract, so it can be detached in an emergency or a migration.
