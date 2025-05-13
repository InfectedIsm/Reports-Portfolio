# Table of Contents

## Medium Severity Issues
- [M01 - `AccessControlHooks::onQueueWithdrawal` do not check `market.isHooked` allowing anyone to call the function with arbitrary `hooksData`](#m01)
- [M02 - `FixedTermLoanHooks` allow borrower to update Annual Interest before end of the "fixed term period"](#m02)
- [M03 - `addRoleProvider` will revert if `providerAddress` is an EOA, while documentation state provider can be an EOA](#m03)
- [M04 - `repayDeliquentDebt` is not effective, as the market will become delinquent again on next block](#m04)

## Low Severity Issues
- [L01 - `WildcatArchController::removeMarket` do not call XSphere.removeAllowedSender](#l01)
- [L02 - `onRepay` hook can be avoided by directly sending funds to market as assets are accounted as balanceOf](#l02)
- [L03 - Market should allow rescuing market's internal tokens as there is no risk of stealing from borrower](#l03)
- [L04 - As `_calculateTemporaryReserveRatioBips` rounds down the `relativeDiff` value, borrower can reduce more than 25% without triggering a Reserve Ratio update](#l04)
- [L05 - Nothing prevent to create an escrow contract after borrower gets sanctionned](#l05)
- [L06 - `getLenderStatus` calls `_tryGetCredential` without checking if `provider` is a pullProvider](#l06)
- [L07 - Sanctioned account can poison market by transfering tokens right before state is updated](#l07)
- [L08 - In `_getUpdateState`, the second `updateScaleFactorAndFees` for non expired batches undervaluate the protocol fees when `_processExpiredWithdrawalBatch` is called right before](#l08)

# 🟡 M01 - `AccessControlHooks::onQueueWithdrawal` do not check `market.isHooked` allowing anyone to call the function with arbitrary `hooksData`
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L812-L825

## Summary
[`AccessControlHooks::onQueueWithdrawal`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L812-L825) has no access control in contrast to all other hooks, making it possible for an attacker to call `_tryValidateAccess(status, lender, hooksData)` with arbitrary `hooksData`.

1. By calling `onQueueWithdrawal` directly, the attacker has access to `_tryValidateAccess` which [can update a lender status](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L705) through `_writeLenderStatus`
2. Thus, any Role Provider listed in `_roleProviders` mapping can be called by the attacker to get credentials, even those who are not in the pull provider list and might be there only to grant access to specific priviledged users.
3. The call is executed by the Hook, meaning the attacker is calling the role provider with arbitrary data with `msg.sender = hook`
4. The attacker can gain access to credentials and operate into the market unexpectedly

## Vulnerability details
If we follow the call path:
1. User call `onQueueWithdrawal` with arbitrary `hooksData` (containing a selected role provider)
2. [`_tryValidateAccess` is called](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L821-L821)
3. Then [`_tryValidateAccessInner` is called](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L704-L704)
4. [`_handleHooksData` is called](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L670-L672) and will return `hasValidCredential == true` if `_tryValidateCredential` return true
5. [`_tryValidateCredential` is called](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L634-L634)
6. And finally the role provider is [extracted from the `hooksData`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L531-L531) and [called](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L553-L561)
7. When the caller successfully get (unauthorized) valid credentials, he can then deposit, [which will this time call `_writeLenderStatus`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L803-L803) with `canSetKnownLender == true`, setting `isKnownLenderOnMarket == true` for that market.
8. Finally, once `isKnownLenderOnMarket == true` for the user, he can transfer market's tokens as the `onTransfer` hook [is skipped](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L863-L863)

**For this to be possible step (7) is crucial, and require the provider to return valid credential to the caller, which is possible depending on the verification method used by that provider. The issue would not exist with an access control on the hook.**

Under in normal circumstances, Lenders have 2 ways to validate their credential:
1. a Role Provider [calls `grantRole`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L376-L382) with the Lender address
2. If Lender do not have valid credentials or valid `lastProvider`, by calling a hooked market function, the hook will go through a list of `PullProvider` to try validate his credentials

Thanks to the missing access control on `onQueueWithdrawal`, a Lender can ask credential to any Provider that are not part of the list above, which might be special Providers that shouldn't be accessed by anyone, and might grant credential to the caller.

## Impact
Unrestricted access to any Provider by a Lender to try validate his credentials.
From the "Attack Ideas" section of the Readme : 
- *"Consider ways in which lenders can receive a credential for a market without 'correctly' passing through the hook instance specified by the borrower that deployed it."*
- *"Consider ways in which access to market interactions can be maliciously altered to either block or elevate parties outside of the defined flow."*

## Tools Used
Manual review

## Recommended Mitigation Steps
```diff
diff --git a/src/access/AccessControlHooks.sol b/src/access/AccessControlHooks.sol
index 1124e5b..bbbabea 100644
--- a/src/access/AccessControlHooks.sol
+++ b/src/access/AccessControlHooks.sol
@@ -816,6 +816,8 @@ contract AccessControlHooks is MarketConstraintHooks {
     MarketState calldata /* state */,
     bytes calldata hooksData
   ) external override {
+	HookedMarket memory market = _hookedMarkets[msg.sender];
+	if (!market.isHooked) revert NotHookedMarket();
	LenderStatus memory status = _lenderStatus[lender];
	if (
		!isKnownLenderOnMarket[lender][msg.sender] && !_tryValidateAccess(status, lender, hooksData)
```

# 🟡 M02 - `FixedTermLoanHooks` allow borrower to update Annual Interest before end of the "fixed term period"
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L857-L859
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L960-L978

## Summary
While the documentation states that in case of 'fixed term' market the APR cannot be changed until the term ends, nothing prevents this in `FixedTermLoanHooks`.

## Vulnerability details
In Wildcat markets, lenders know in advance how much `APR` the borrower will pay them. In order to allow lenders to exit the market swiftly, the market must always have at least a `reserve ratio` of the lender funds ready to be withdrawn.
If the borrower decide to [reduce the `APR`](https://docs.wildcat.finance/using-wildcat/day-to-day-usage/borrowers#reducing-apr), in order to allow lenders to 'ragequit', a new `reserve ratio` is calculated based on the variation of the APR as described in the link above.
Finally, is a market implement a fixed term (date until when withdrawals are not possible), it shouldn't be able to reduce the APR, as this would allow the borrower to 'rug' the lenders by reducing the APR to 0% while they couldn't do anything against that.

The issue here is that while lenders are (as expected) prevented to withdraw before end of term:
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L857-L859
```solidity
File: src/access/FixedTermLoanHooks.sol
848:   function onQueueWithdrawal(
849:     address lender,
850:     uint32 /* expiry */,
851:     uint /* scaledAmount */,
852:     MarketState calldata /* state */,
853:     bytes calldata hooksData
854:   ) external override {
855:     HookedMarket memory market = _hookedMarkets[msg.sender];
856:     if (!market.isHooked) revert NotHookedMarket();
857:     if (market.fixedTermEndTime > block.timestamp) {
858:       revert WithdrawBeforeTermEnd();
859:     }
```

this is not the case for the borrower setting the annual interest:
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L960-L978
```solidity
File: src/access/FixedTermLoanHooks.sol
960:   function onSetAnnualInterestAndReserveRatioBips(
961:     uint16 annualInterestBips,
962:     uint16 reserveRatioBips,
963:     MarketState calldata intermediateState,
964:     bytes calldata hooksData
965:   )
966:     public
967:     virtual
968:     override
969:     returns (uint16 updatedAnnualInterestBips, uint16 updatedReserveRatioBips)
970:   {
971:     return
972:       super.onSetAnnualInterestAndReserveRatioBips(
973:         annualInterestBips,
974:         reserveRatioBips,
975:         intermediateState,
976:         hooksData
977:       );
978:   }
979: 
```

## Impact
Borrower can rug the lenders by reducing the APR while they cannot quit the market

## Proof of Concept

Add this test to `test/access/FixedTermLoanHooks.t.sol`

```c
	  function testAudit_SetAnnualInterestBeforeTermEnd() external {
	    DeployMarketInputs memory inputs;

		// "deploying" a market with MockFixedTermLoanHooks
		inputs.hooks = EmptyHooksConfig.setHooksAddress(address(hooks));
		hooks.onCreateMarket(
			address(this),				// deployer
			address(1),				// dummy market address
			inputs,					// ...
			abi.encode(block.timestamp + 365 days, 0) // fixedTermEndTime: 1 year, minimumDeposit: 0
		);

		vm.prank(address(1));
		MarketState memory state;
		// as the fixedTermEndTime isn't past yet, it's not possible to withdraw
		vm.expectRevert(FixedTermLoanHooks.WithdrawBeforeTermEnd.selector);
		hooks.onQueueWithdrawal(address(1), 0, 1, state, '');

		// but it is still possible to reduce the APR to zero
		hooks.onSetAnnualInterestAndReserveRatioBips(0, 0, state, "");
	  }
```

## Tools Used
Manual review

## Recommended Mitigation Steps

When `FixedTermLoanHooks::onSetAnnualInterestAndReserveRatioBips` is called, revert if `market.fixedTermEndTime > block.timestamp`


____________________________________
# 🟡 M03 - `addRoleProvider` will revert if `providerAddress` is an EOA, while documentation state provider can be an EOA
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L220-L220
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L257-L257

## Summary
`addRoleProvider` will revert if `providerAddress` is an EOA, while documentation state provider can be an EOA.

## Vulnerability details
Role providers can grant credentials to lenders by calling the `grantRole` function.

The project [documentation describe](https://docs.wildcat.finance/technical-overview/security-developer-dives/hooks/access-control-hooks#role-providers) the role provider interface, and says:
> Role providers can "push" credentials to the hooks contract by calling `grantRole`:
> \[...\]
> **Role providers do not have to implement any of these functions - a role provider can be an EOA.**

But if we try to set an EOA as a `providerAddress` when calling `addRoleProvider`, the call will revert as `isPullProvider` is not implemented on `providerAddress` (which is an EOA).


```solidity
File: src/access/AccessControlHooks.sol
217:   function addRoleProvider(address providerAddress, uint32 timeToLive) external onlyBorrower {
218:     RoleProvider provider = _roleProviders[providerAddress];
219:     if (provider.isNull()) {
220:❌    bool isPullProvider = IRoleProvider(providerAddress).isPullProvider();
221:       // Role providers that are not pull providers have `pullProviderIndex` set to
222:       // `NotPullProviderIndex` (max uint24) to indicate they do not refresh credentials.
223:       provider = encodeRoleProvider(
224:         timeToLive,
225:         providerAddress,
226:         isPullProvider ? uint24(_pullProviders.length) : NotPullProviderIndex
227:       );
```

Also, `_tryValidateCredential` throw error if `validateCredential` return data < 32 bites, which means that if last provider for a user was an EOA, he will revert before being able to go through the pullProviders list
Might want to cover this case by checking if address has code too


## Impact
EOA cannot be role providers, breaking a core functionality of the protocol.

## Tools Used
Manual review

## Recommended Mitigation Steps
Perform a low level call to cover the case where the providerAddress has no code.
```diff
diff --git a/src/access/AccessControlHooks.sol b/src/access/AccessControlHooks.sol
index 1124e5b..0fbc0d9 100644
--- a/src/access/AccessControlHooks.sol
+++ b/src/access/AccessControlHooks.sol
@@ -217,7 +217,13 @@ contract AccessControlHooks is MarketConstraintHooks {
   function addRoleProvider(address providerAddress, uint32 timeToLive) external onlyBorrower {
     RoleProvider provider = _roleProviders[providerAddress];
     if (provider.isNull()) {
-      bool isPullProvider = IRoleProvider(providerAddress).isPullProvider();
+                       bool isPullProvider;
+                       (bool success, bytes memory data) = providerAddress.call(
+                               abi.encodeWithSelector(IRoleProvider.isPullProvider.selector, "")
+                       );
+                       if(success && data.length == 32){
+                                       isPullProvider = abi.decode(data, (bool));
+                       }
       // Role providers that are not pull providers have `pullProviderIndex` set to
       // `NotPullProviderIndex` (max uint24) to indicate they do not refresh credentials.
       provider = encodeRoleProvider(
```

# 🟡 M04 - `repayDeliquentDebt` is not effective, as the market will become delinquent again on next block

## Summary
`repayDelinquentDebt` allow a borrower to repay the exact amount necessary to cover the delinquency, i.e make the `totalAssets` in the market equal to the `liquidityRequired`. 
But as fees accrues every seconds (and new block), the market will be delinquent again on next block.

## Vulnerability details

The `repayDelinquentDebt` works that way:
1. The market state is updated in `_getUpdatedState()`. During this call, as the market is delinquent, delinquency is applied (either by applying fees or increasing the delinquent timer) 
2. Then the exact amount to make the market not delinquent is computed as `delinquentDebt`
3. [The amount is repaid](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L172-L172) (a transfer of the `delinquentDebt` is made from the borrower to the market)
4. Finally, the `state.isDelinquent` variable is [set to false](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L541-L542) in `_writeState` as `state.liquidityRequired() == totalAssets()` now.

```solidity
File: src/market/WildcatMarket.sol
186:   function repayDelinquentDebt() external nonReentrant sphereXGuardExternal {
187:     MarketState memory state = _getUpdatedState();
188:     uint256 delinquentDebt = state.liquidityRequired().satSub(totalAssets());
189:     _repay(state, delinquentDebt, 0x04);
190:     _writeState(state);
191:   }
```

```solidity
File: src/market/WildcatMarketBase.sol
540:   function _writeState(MarketState memory state) internal {
541:     bool isDelinquent = state.liquidityRequired() > totalAssets();
542:     state.isDelinquent = isDelinquent; 
543:	// .... 
544: // ....	
```

The issue here, is that on the next block that will occur, any stateful interaction (starts with `_getUpdatedState()` and ends with `_writeState`) with the market [will increase the fees and reserve ratio](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/libraries/MarketState.sol#L87-L98) in `_getUpdatedState()`, increasing `state.liquidityRequired()`, making the market delinquent again when `_writeState` is called at the end.

## Impact
This will cause the market to still accrue delinquency fees, causing a loss for the borrower.
The `repayDelinquentDebt` purpose is defeated by the fact this only remove the delinquency of the market for one block, which is not effective to help a borrower solve the issue effectively, who will see the delinquency continue to affect its market, causing additional cost that could have been avoided by adding a buffer of repayment.

## Proof of Concept
Add this test to `test/market/WildcatMarket.t.sol`:

```solidity
  function testAudit_repayDelinquentDebtNotEffective() external {
    _depositBorrowWithdraw(alice, 1e18, 8e17, 2e17); // 20% of 8e17
    asset.mint(address(this), 1.6e17);
    asset.approve(address(market), 1.6e17);
		
	// market is deliquent
	MarketState memory state = market.currentState();
	assertEq(true, state.isDelinquent);

		// borrower repay the delinquecy
    market.repayDelinquentDebt();
		
		// market is no more deliquent as it has been repaid
	market.updateState();
	state = market.currentState();
	assertEq(false, state.isDelinquent);

	console.log("--- state after repayDelinquentDebt ---");
	console.log("---------------------------------------");
	console.log("isDelinquent: ", state.isDelinquent);
	console.log("totalAssets: %e", market.totalAssets());
	console.log("liquidityRequired: %e\n", state.liquidityRequired());

	// increasing timestamp by 12s to simulate one new block 
	vm.warp(block.timestamp + 12);

	// as exected, market is deliquent as even 1 wei of added interest will make the market delinquent again
	market.updateState();
	state = market.currentState();
	assertEq(true, state.isDelinquent);

	console.log("---  state after one block elapsed  ---");
	console.log("---------------------------------------");
	console.log("isDelinquent: ", state.isDelinquent);
	console.log("totalAssets: %e", market.totalAssets());
	console.log("liquidityRequired: %e", state.liquidityRequired());
  }

```

Ouput for the test:
```c
Ran 1 test for test/market/WildcatMarket.t.sol:WildcatMarketTest
[PASS] testAudit_repayDelinquentDebtNotEffective() (gas: 1070934)
Logs:
  computeCreateAddress is deprecated. Please use vm.computeCreateAddress instead.
  --- state after repayDelinquentDebt ---
  ---------------------------------------
  isDelinquent:  false
  totalAssets: 3.6e17
  liquidityRequired: 3.6e17

  ---  state after one block elapsed  ---
  ---------------------------------------
  isDelinquent:  true
  totalAssets: 3.6e17
  liquidityRequired: 3.60000009132420091e17
```

## Tools Used
Manual review

## Recommended Mitigation Steps
The goal is to give more time such that the `timeDelinquent` timer can decrease when the delinquency is repaid, and allow the borrower to act.
I see 2 solutions:
1. Add a fixed repayment buffer to cover for the increase in fees on next blocks
2. Add an input value allowing the to the amount of buffer he wants to add 

This would also cover the [1-2 wei corner case](https://github.com/lidofinance/lido-dao/issues/442) of stETH (a rebasing token) in case a market uses it as its asset.


____________________________________

# Q/A and Low findings

# 🔵L01 - `WildcatArchController::removeMarket` do not call XSphere.removeAllowedSender
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatArchController.sol#L347-L360

## Vulnerability details
Adding a market to the Arch Controller also add its address to the allowed sender of the XSphere component.
But removing the market from the Arch Controller do not remove it from the allowed senders

```solidity
File: src/WildcatArchController.sol
347:   function registerMarket(address market) external onlyController {
348:     if (!_markets.add(market)) {
349:       revert MarketAlreadyExists();
350:     }
351:     _addAllowedSenderOnChain(market);
352:     emit MarketAdded(msg.sender, market);
353:   }
354: 
355:   function removeMarket(address market) external onlyOwner {
356:     if (!_markets.remove(market)) {
357:       revert MarketDoesNotExist();
358:     }
359:     emit MarketRemoved(market);
360:   }

```

## Impact
If a market behave maliciously and is removed from the Arch Controller, it will still be considered a trustable address for the XSphere defensive mechanism.

## Tools Used
Manual review

## Recommended Mitigation Steps
Call `removeAllowedSender` : https://github.com/spherex-xyz/spherex-contracts/blob/f025ca3fcdc3705c0d87a0b198a2cf762bca7310/src/SphereXEngine.sol#L161

____________________________________
# 🔵L02 - `onRepay` hook can be avoided by directly sending funds to market as assets are accounted as balanceOf
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L212-L212

## Summary
Because `market.totalAssets()` uses `asset.balanceOf()` to compute the available assets to execute withdrawals, it is possible to repay the market without going through the `onRepay` function, which might be important.

## Vulnerability details
Right now, none of the developed hooks implemented a logic for `onRepay` so there is no impact.
But once this hook will be used, please note that the borrower will be able to possibly not be subject to the necessary hook logic by simply transfering the repay amount directly to the contract.

This is the case 

## Impact
Borrower can avoid executing the `onRepay` logic.

## Tools Used
Manual review

## Recommended Mitigation Steps
Using an internal accounting for `totalAssets` would be a solution, but this would  make rebasing tokens incompatible.

____________________________________
# 🔵L03 - Market should allow rescuing market's internal tokens as there is no risk of stealing from borrower
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L38-L38

## Vulnerability details
The `WildcatMarket::rescueToken` function prevent borrower to rescue market's asset tokens, which make total sense as this would allow to borrow the funds without going through the borrower Chainalysis sanction check.

But the function disallow the transfer of market's shares, which make seems to make less sense, as the market should never have a balance of its own tokens in the first place, as only Accounts that deposited, or got transfered market's shares have a balance.

## Impact
If non tech-savy lenders mistakenly transfer market's shares to the market (which is likely to happen as this can be seen in many DeFi protocols), there is no way to recover them, while this is the purpose of the function.

## Tools Used
Manual review

## Recommended Mitigation Steps
To keep the `rescueToken` unchanged, a check could be added in `WildcatMarketToken::transfer` if the recipient `to` is the market itself.


____________________________________
# 🔵 L04 - As `_calculateTemporaryReserveRatioBips` rounds down the `relativeDiff` value, borrower can reduce more than 25% without triggering a Reserve Ratio update
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/MarketConstraintHooks.sol#L167-L167

## Vulnerability details
The `_calculateTemporaryReserveRatioBips` checks if the APR has been reduced by more than 25% from its previous value.
[If this is the case](https://docs.wildcat.finance/using-wildcat/day-to-day-usage/borrowers#reducing-apr), the Reserve Ratio of the market must be increased based on a formula
As the precision of the calculations are Bips (10_000), if the borrower reduce the APR by 25.009%, this will be considered 25.00% still, and will not trigger a Reserve Ratio update.

## Impact
Borrower can reduce the APR more than it should without triggering the Reserve Ratio update.

## Tools Used
Manual review

## Recommended Mitigation Steps
The `relativeDiff` calculation should use the `bipMul` function which proceed with a half-rounding, rather than always rounding down.


____________________________________
# 🔵 L05 - Nothing prevent to create an escrow contract after borrower gets sanctionned

## Vulnerability details
https://docs.wildcat.finance/using-wildcat/day-to-day-usage/the-sentinel#borrower-gets-sanctioned
The Wildcat Documentation states that once a borrower get sanctionned no escrow contract can be created for the borrower's related markets.
But this is not true, right now, nothing prevent an escrow contract to be created:
- `WildcatMarketWithdrawals::executeWithdrawal` do not check borrower's sanction before creating the escrow contract
- the `WildcatSanctionsSentinel::createEscrow` function is not protected and allow anyone to create an escrow contract

## Impact
What seems a requirement from the specification is not respected by the implementation.

## Proof of Concept
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L194-L194
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L121-L121

## Tools Used
Manual review

## Recommended Mitigation Steps
check borrower's sanction before creating the escrow contract.


____________________________________
# 🔵 L06 - `getLenderStatus` calls `_tryGetCredential` without checking if `provider` is a pullProvider

## Summary
`_tryGetCredential` should only be called if a provider is a pull provider [as explained in the function natspec](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L467-L468), but `getLenderStatus` do not perform this check thoroughly.

## Vulnerability details
Before calling `_tryGetCredential`, `getLenderStatus` does a check on the lender status to verify if the previous:
```solidity
File: src/access/AccessControlHooks.sol
344:         if (status.canRefresh) {
345:           if (_tryGetCredential(status, provider, accountAddress)) {
346:             return status;
347:           }
```

To inform they are pull providers (can automatically refresh credential), role providers must implement [a function `isPullProvider()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/IRoleProvider.sol#L5)
The implementation is not define, and do not say if the function will always return 'true', as it might depend on an internal logic.

That means that it is possible that a role provider who stop becoming a pull provider for any reason will still be accessed through the function to get the lender status and tell if his credential are valid. 

## Impact
`_tryGetCredential` called without checking if the provider is still a pull provider

## Tools Used
Manual review

## Recommended Mitigation Steps
Call the provider address with `isPullProvider()` to ensure it can still grant credentials

____________________________________
# 🔵 L07 - Sanctioned account can poison market by transfering tokens right before state is updated.
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L458

## Vulnerability details
A malicious actor with sanctionned funds can send tokens to a market right before `updateState/_getUpdatedState` and `_writeState` are called. The funds will be marked for a batch, and registered in the `availableLiquidity`

Then, it will be used to calculate the new values for `batch.scaledAmountBurned` and `state.scaledPendingWithdrawals`, basically poisoning the market and lenders who received those assets with assets from an OFAC sanctionned address.
This could cause multiple lenders to be also flagged as sanctionned causing further legal issues.

Because assets can be deposited to be distributed without interacting with the protocol external functions, XSphere protection will have no power to block this.
Also, this might not be noticed by the borrower.

## Impact
Market can distribute sanctionned funds to lenders which might cause trouble to borrower and all users

## Tools Used
Manual review

## Recommended Mitigation Steps
Use an internal accounting system to ensure no poisoned funds can be dispersed through the market.

____________________________________
# 🔵 L08 - In `_getUpdateState`, the second `updateScaleFactorAndFees` for non expired batches undervaluate the protocol fees when `_processExpiredWithdrawalBatch` is called right before

## Vulnerability details
See similar finding: https://github.com/code-423n4/2023-10-wildcat-findings/issues/455

Every time `_getUpdateState` is called, two fees update are computed following this scheme:
1. if expired batches exists, compute fees for expired batch and update `lastInterestAccruedTimestamp = expiry`
2. process expired batch, which update the `state.scaledTotalSupply` value (reducing it)
3. compute fees from `lastInterestAccruedTimestamp` to `block.timestamp`

The thing is that `updateScaleFactorAndFees` uses `state.scaledTotalSupply` [to compute the protocol fees](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/libraries/FeeMath.sol#L40-L50), but as the `state.scaledTotalSupply` is reduced in step (2), the amount of fees paid for the period `block.timestamp - expiry` is computed on the `state.scaledTotalSupply` **minus what has been processed in `_processExpiredWithdrawalBatch`**, even though that the `state.scaledTotalSupply` was present in the market until `block.timestamp`.

## Impact
Wrong computation of the scale factor and fees.

## Tools Used
Manual review
