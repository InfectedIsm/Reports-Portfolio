## H-01 OFAC sanctioned lender can frontrun nukeFromOrbit with a transfer of his funds

# Lines of code
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74-L81
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L163
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L64-L82
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L152-L153


# Vulnerability details
## Impact
In order to prevent a sanctioned lender (for example by OFAC) to poison an entire market, a function has been developed to block and transfer the sanctionned user's funds to an escrow contract. This escrow contract can be released if borrower decides so (by setting up an override), or if sanctioned lender is removed from the sanction lists.
To do that, the borrower (or any actor) can call [`nukeFromOrbit`](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74-L81), which will change authentication role of user to `AuthRole.Blocked` which basically disallow him any action on the market, and as said before, by calling `_blockAccount` [seize and transfer the funds to an escrow contract](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L170-L183), waiting for more clarity on the situation.
But there is a way for the sanctionned lender to avoid this situation.

The requirement are:
1) a non-zero scaledBalance (cannot be applied if already in queue)
2) a second authorized lender address in the market (as borrower is responsible for the verification, he could be tricked)

## Proof of Concept

1) sanctionned lender monitor the mempool for future call to [`nukeFromOrbit`](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketConfig.sol#L74-L81)
2) sanctioned lender see the call and transfer its scaledBalance to the 2nd address by calling `[WildcatMarketToken::transfer](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L36-L39)`
3) [Even if `_transfer`](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketToken.sol#L64-L82), through `_getAccount` [ensure lender isn't blocked](https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L152-L153), this check pass as he is not blocked yet, only sanctionned
4) using the 2nd address he can either:
        a. wait (only sanction list make it possible to block an account, and only blocked account balances can be escrowed)
        b. or directly queue the scaledBalance into the current batch

Once the balance has been queued, while borrower could decide not to process the request (to prevent user to get the funds), this would have this impact:
1) other lenders that have queued after/inside this batch will not be able to withdraw until then
2)  If the sanctionned lender make a withdrawal request putting the market into deliquency, this will either force borrower to solve the deliquency, or incure the penalty. And by solving the deliquency, it will allow the sanctionned lender to exit the market with the funds.

All of this is possible because nothing in `WildcatMarketToken` prevent a sanctioned lender to operate a transfer, even if he is in the sanction list.
There is indeed a `_getAccount` call which checks if an address has `AuthRole.Blocked` and revert if true, but by front-running the blocking action, he can transfer the funds before this role is assigned.

Because this situation will impact the withdrawing function availability until solved, and can incur losses (through deliquency) to the borrower until finding a proper solution, also impacting the reputation of the protocol, I categorized this vulnerabilty as high.

## Tools Used
Code review

## Recommended Mitigation Steps
It doesn't seem that transfer of scaledBalance will be used very often, devs could add a check to sanction list on the "from" address to prevent a sanctionned user to move its funds at all, and thus escape the escrow.
This change could be applied:

```diff
diff --git a/src/market/WildcatMarketToken.sol b/src/market/WildcatMarketToken.sol
index f6a7f94..27c32c8 100644
--- a/src/market/WildcatMarketToken.sol
+++ b/src/market/WildcatMarketToken.sol
@@ -62,6 +62,7 @@ contract WildcatMarketToken is WildcatMarketBase {
  }

  function _transfer(address from, address to, uint256 amount) internal virtual {
+    require(!IWildcatSanctionsSentinel(sentinel).isSanctioned(borrower, from),"from is sanctionned");
    MarketState memory state = _getUpdatedState();
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();
```


## Assessed type
Invalid Validation



## M-01 Sanctionned funds keep earning APR, and protocol earning fees on these funds
# Lines of code
https://github.com/code-423n4/2023-10-wildcat/blob/c5df665f0bc2ca5df6f06938d66494b11e7bdada/src/market/WildcatMarketBase.sol#L163

# Vulnerability details
## Impact
When a user is sanctioned, if he has a `scaledBalance` (not in the withdrawal queue), calling the `nukeFromOrbit` function will send sanctioned funds to an escrow contract, and these funds will keep earning APR.

This is because when a deposit is executed, the deposited amount is converted to a `scaledBalance`, by dividing amount by the `scaleFactor` (which can be seen as an exchange rate between the underlying token and the market shares). And only when withdrawn, the `scaleBalance` is multiplied back by the scaleFactor, which will have grown in regard to the APR and other fees owed by the borrower.

But the protocol itself will also earn fees on the sanctionned balance, as the `ProtocolFee` is calculated from `state.scaledTotalSupply` (L47):
```solidity
File: src\libraries\FeeMath.sol
40:   function applyProtocolFee(
41:     MarketState memory state,
42:     uint256 baseInterestRay,
43:     uint256 protocolFeeBips
44:   ) internal pure returns (uint256 protocolFee) {
45:     // Protocol fee is charged in addition to the interest paid to lenders. //note baseInterestRay: interest rate accrued to lenders
46:     uint256 protocolFeeRay = protocolFeeBips.bipMul(baseInterestRay);
47:     protocolFee = uint256(state.scaledTotalSupply).rayMul(
48:       uint256(state.scaleFactor).rayMul(protocolFeeRay)
49:     );
50:     state.accruedProtocolFees = (state.accruedProtocolFees + protocolFee).toUint128();
51:   }
```

And this variable isn't decreased when escrowing the sanctionned funds.
This means both borrower and protocol will still interact with the sanctionned address and funds, even if it has been blocked by the market.

## Proof of Concept

## Tools Used
Code review

## Recommended Mitigation Steps
When blocking an account, automatically queue, apply and execute sanctionned lender balance so that it is normalized back, removed from the total supply and thus stop earning fees and paying APR on it.
This will ensure that no sanctioned funds leaks after being blocked.

Another way to do this would be to save the `scaleFactor` in a specific variable (like `scaleFactorAtSanction`) and use this scale factor rather than the current one to convert the market shares to the underlying asset on withdraw.


## Assessed type

Other




