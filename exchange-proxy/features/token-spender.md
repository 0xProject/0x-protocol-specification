# TokenSpender

We need a way to transfer funds from the taker into the Exchange proxy temporarily so we can perform transformations on it. We could simply have the taker set an allowance for the Exchange proxy, but this comes with some risk:

* Lack of separation of complex logic vs access to funds.
* No easy way to detach allowances in case of a critical vulnerability.
* Allowances do not migrate if we ever redeploy the Exchange proxy.
* Dangling allowances if we decide to migrate the Exchange proxy to use the V3 asset proxies for allowances.

So we opt for something akin to the role of asset proxies in V3, where takers must set their allowance to a separate, specialized spender contract accessible by the Exchange proxy. However, instead of having a single allowance contract for each asset type, we instead have a **single**, universal spender contract. Takers can get the address of this contract with `getAllowanceTarget()`.

This universal allowance contract has a single `executeCall()` function which blindly `call`s calldata on the caller’s (Exchange Proxy) behalf. This contract is akin the `FlashWallet` used by `transformERC20()` with some notable differences:

* Implements the `Authorizable` mixin.
    * Owner will be set to the governor.
    * The `ZeroEx` contract will be the sole authorized address.
* Cannot perform a `delegatecall`. Instead exposes an (authority-only) `executeCall()` function that simply forwards a call.
* Is never intended to hold a balance.

Through `executeCall()`, the Exchange proxy can call `transferFrom()` in the `AllowanceTarget`’s context to move taker funds. A dedicated feature, `TokenSpender`, will wrap this functionality with convenience functions such as `_spendERC20Tokens(token, from to, amount)`.

![token-spender](https://raw.githubusercontent.com/0xProject/0x-protocol-specification/master/exchange-proxy/img/token-spender.png)

The `AllowanceTarget` instance is owned by the governor contract, so it can be detached in an emergency or a migration.
