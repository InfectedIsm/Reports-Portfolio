# Venus Protocol (May 2023)

# Low Risk and Non-Critical Issues

## L-01 maxLoopsLimit can only be increased

`Comptroller`, `RewardDistributor` and `RiskFund` all inherit from `MaxLoopsLimitHelper`, which work is to ensure that loops cannot iterate more than a configured value. It is intended (as stated by its documentation) to avoid DoS.

But in case too much gas is consumed for N loops, the only way to reduce the gas consumption is to reduce the number of loops.

It should be posible to either increase or decrease the limit depending on the situation.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/MaxLoopsLimitHelper.sol#L25-L32

```solidity
File: contracts\MaxLoopsLimitHelper.sol
25:     function _setMaxLoopsLimit(uint256 limit) internal {
26:         require(limit > maxLoopsLimit, "Comptroller: Invalid maxLoopsLimit");
27:
28:         uint256 oldMaxLoopsLimit = maxLoopsLimit;
29:         maxLoopsLimit = limit;
30:
31:         emit MaxLoopsLimitUpdated(oldMaxLoopsLimit, maxLoopsLimit);
32:     }

```

## L-02 initializeMarket update `supplyState.block` and `borrowState.block` even if it does not update `supplyState.index` and `borrowState.index`

The function `initializeMarket` can also be called on already initialized markets. If executed, block number is updated while index is not, which does not reflect what is stated in the declaration of the related mapping and struct:

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Rewards/RewardsDistributor.sol#L16-L23

```solidity
File: contracts\Rewards\RewardsDistributor.sol15:     struct RewardToken {
16:         // The market's last updated rewardTokenBorrowIndex or rewardTokenSupplyIndex
17:         uint224 index;
18:         **// The block number the index was last updated at**
19:         uint32 block;
20:     }
21:
22:     /// @notice The REWARD TOKEN market supply state for each market
23:     mapping(address => RewardToken) public rewardTokenSupplyState;

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Rewards/RewardsDistributor.sol#L125-L150

```solidity
File: contracts\Rewards\RewardsDistributor.sol

125:     function initializeMarket(address vToken) external onlyComptroller {
126:         uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
127:
128:         RewardToken storage supplyState = rewardTokenSupplyState[vToken];
129:         RewardToken storage borrowState = rewardTokenBorrowState[vToken];
130:
131:         /*
132:          * Update market state indices
133:          */
134:         if (supplyState.index == 0) {
135:             // Initialize supply state index with default value
136:             supplyState.index = rewardTokenInitialIndex;
137:         }
138:
139:         if (borrowState.index == 0) {
140:             // Initialize borrow state index with default value
141:             borrowState.index = rewardTokenInitialIndex;
142:         }
143:
144:         /*
145:          * Update market state block numbers
146:          */
147:         supplyState.block = borrowState.block = blockNumber;
148:
149:         emit MarketInitialized(vToken);
150:     }

```

## L-03 - Missing role access control on `reduceReserves` in `VToken` contract

reduceReserves can be called to send the actual vToken reserves to the Protocol Share Reserve. This kind of action should probably be restricted to specific roles and not accessible to anyone.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken.sol#L337-L348

```solidity
File: contracts\VToken.sol
337:     /**
338:      * @notice Accrues interest and reduces reserves by transferring to the protocol reserve contract
339:      * @param reduceAmount Amount of reduction to reserves
340:      * @custom:event Emits ReservesReduced event; may emit AccrueInterest
341:      * @custom:error ReduceReservesCashNotAvailable is thrown when the vToken does not have sufficient cash
342:      * @custom:error ReduceReservesCashValidation is thrown when trying to withdraw more cash than the reserves have
343:      * @custom:access Not restricted
344:      */
345:     function reduceReserves(uint256 reduceAmount) external override nonReentrant { //@audit L05 - Missing role access control
346:         accrueInterest();
347:         _reduceReservesFresh(reduceAmount); //@audit-ok N04 - discrepancy with doc: reduceReserves do not send all, but only reduceAmount
348:     }

```

## L-04 - `oracle.getUnderlyingPrice` should always be preceded by `oracle.updatePrice`

Oracles in Venus are based on ResilientOracle (if what was shared in channel was correct) from this repo : https://github.com/VenusProtocol/oracle/blob/develop/contracts/ResilientOracle.sol

The `updatePrice` function comment states that it should always be called before calling getUnderlyingPrice. While this is mostly the case in the code, there are some exceptions:

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L755-L757

```solidity
File: contracts\Comptroller.sol
755:         if (newCollateralFactorMantissa != 0 && oracle.getUnderlyingPrice(address(vToken)) == 0) { //@audit ? missing updatePrice?
756:             revert PriceError(address(vToken));
757:         }

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Lens/PoolLens.sol#L266-L271

```solidity
File: contracts\Lens\PoolLens.sol
263:         for (uint256 i; i < markets.length; ++i) {
264:             BadDebt memory badDebt;
265:             badDebt.vTokenAddress = address(markets[i]);
266:             badDebt.badDebtUsd =
267:                 VToken(address(markets[i])).badDebt() *
268:                 priceOracle.getUnderlyingPrice(address(markets[i])); //@audit - L04 Missing call to updatePrice, meaning badDebtUsd can be wrong
269:             badDebtSummary.badDebts[i] = badDebt;
270:             totalBadDebtUsd = totalBadDebtUsd + badDebt.badDebtUsd;
271:         }

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Lens/PoolLens.sol#L401-L410

```solidity
File: contracts\Lens\PoolLens.sol
401:     function vTokenUnderlyingPrice(VToken vToken) public view returns (VTokenUnderlyingPrice memory) {
402:         ComptrollerViewInterface comptroller = ComptrollerViewInterface(address(vToken.comptroller()));
403:         PriceOracle priceOracle = comptroller.oracle();
404:
405:         return
406:             VTokenUnderlyingPrice({
407:                 vToken: address(vToken),
408:                 underlyingPrice: priceOracle.getUnderlyingPrice(address(vToken)) //@audit ? missing updatePrice?
409:             });
410:     }

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L1368-L1374

```solidity
File: contracts\Comptroller.sol
1368:     function _safeGetUnderlyingPrice(VToken asset) internal view returns (uint256) {
1369:         uint256 oraclePriceMantissa = oracle.getUnderlyingPrice(address(asset));
1370:         if (oraclePriceMantissa == 0) {
1371:             revert PriceError(address(asset));
1372:         }
1373:         return oraclePriceMantissa;
1374:     }

```

---

## N-01 Ownable2StepUpgradeable already inherited by AccessControlledV8

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L85-L86

```solidity
File: contracts\RiskFund\RiskFund.sol
19: contract RiskFund is
20:     Ownable2StepUpgradeable,
21:     AccessControlledV8,
22:     ExponentialNoError,
23:     ReserveHelpers,
24:     MaxLoopsLimitHelper,
25:     IRiskFund
26: {
	//...
85:         __Ownable2Step_init();
86:         __AccessControlled_init_unchained(accessControlManager_); //should directly call __AccessControlled_init

```

## N-02 Missing natspecs parameters in some functions

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L145-L155

```solidity
File: contracts\RiskFund\RiskFund.sol
145:     /**
146:      * @notice Swap array of pool assets into base asset's tokens of at least a mininum amount.
147:      * @param markets Array of vTokens whose assets to swap for base asset     //should also specify first path as market and last path as base asset
148:      * @param amountsOutMin Minimum amount to recieve for swap
149:      * @return Number of swapped tokens.
150:      */
151:     function swapPoolsAssets(
152:         address[] calldata markets,
153:         uint256[] calldata amountsOutMin,
154:         address[][] calldata paths
155:     ) external override returns (uint256) {

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L225-L230

```solidity
File: contracts\RiskFund\RiskFund.sol
218:     /**
219:      * @dev Swap single asset to base asset.
220:      * @param vToken VToken
221:      * @param comptroller Comptroller address
222:      * @param amountOutMin Minimum amount to receive for swap
223:      * @return Number of swapped tokens.
224:      */
225:     function _swapAsset(
226:         VToken vToken,
227:         address comptroller,
228:         uint256 amountOutMin,
229:         address[] calldata path
230:     ) internal returns (uint256) {

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L369-L381

```solidity
File: contracts\Rewards\RewardsDistributor.sol
369:     /**
370:      * @notice Calculate reward token accrued by a borrower and possibly transfer it to them.
371:      * @param vToken The market in which the borrower is interacting
372:      * @param borrower The address of the borrower to distribute REWARD TOKEN to
373:      */
374:     function _distributeBorrowerRewardToken(
375:         address vToken,
376:         address borrower,
377:         Exp memory marketBorrowIndex
378:     ) internal {
379:         RewardToken storage borrowState = rewardTokenBorrowState[vToken];
380:         uint256 borrowIndex = borrowState.index;
381:         uint256 borrowerIndex = rewardTokenBorrowerIndex[vToken][borrower];

```

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L60-L70

```solidity
File: contracts\RiskFund\ProtocolShareReserve.sol
60:     /**
61:      * @dev Release funds
62:      * @param asset  Asset to be released
63:      * @param amount Amount to release
64:      * @return Number of total released tokens
65:      */
66:     function releaseFunds(
67:         address comptroller,
68:         address asset,
69:         uint256 amount
70:     ) external returns (uint256) {

```

## N-03 Discrepancy with documentation: reduceReserves do not send all the reserve but only reduceAmount

Documentation states : *"When reduceReserves() is called in a vToken contract, all accumulated liquidation fees and spread are sent to the ProtocolShareReserve contract"*

But in the code, sent amount can be chosen:

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/VToken/VToken.sol#L345-L347

```solidity
File: contracts\VToken.sol
345:     function reduceReserves(uint256 reduceAmount) external override nonReentrant {
346:         accrueInterest();
347:         _reduceReservesFresh(reduceAmount);
348:     }

```

Documentation should be updated to reflect this.

## N-04 Consider adding an event entry for each effectivly swapped asset

In RiskFund::swapPoolsAssets function, an event is emitted at the end of the function, but it does not specify which assets were swapped.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L179

```solidity
File: contracts\RiskFund\RiskFund.sol
179:         emit SwappedPoolsAssets(markets, amountsOutMin, totalAmount);

```

In _swapAssets, if the amount in USD of tokens to swap is not enough, no swap is done. It would be interesting to know which assets were swapped and which were not if any investigation must be done for any reason.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/RiskFund/RiskFund.sol#L244-L274

```solidity
File: contracts\RiskFund\RiskFund.sol
244:         if (balanceOfUnderlyingAsset > 0) {
245:             Exp memory oraclePrice = Exp({ mantissa: underlyingAssetPrice });
246:             uint256 amountInUsd = mul_ScalarTruncate(oraclePrice, balanceOfUnderlyingAsset);
247:
248:             if (amountInUsd >= minAmountToConvert) {
249:                 assetsReserves[underlyingAsset] -= balanceOfUnderlyingAsset;
250:                 poolsAssetsReserves[comptroller][underlyingAsset] -= balanceOfUnderlyingAsset;
251:
252:                 if (underlyingAsset != convertibleBaseAsset) {
253:                     require(path[0] == underlyingAsset, "RiskFund: swap path must start with the underlying asset");
254:                     require(
255:                         path[path.length - 1] == convertibleBaseAsset,
256:                         "RiskFund: finally path must be convertible base asset"
257:                     );
258:                     IERC20Upgradeable(underlyingAsset).safeApprove(pancakeSwapRouter, 0);
259:                     IERC20Upgradeable(underlyingAsset).safeApprove(pancakeSwapRouter, balanceOfUnderlyingAsset);
260:                     uint256[] memory amounts = IPancakeswapV2Router(pancakeSwapRouter).swapExactTokensForTokens(
261:                         balanceOfUnderlyingAsset,
262:                         amountOutMin,
263:                         path,
264:                         address(this),
265:                         block.timestamp
266:                     );
267:                     totalAmount = amounts[path.length - 1];
268:                 } else {
269:                     totalAmount = balanceOfUnderlyingAsset;
270:                 }
271:             }
272:         }
273:
274:         return totalAmount;

```

## N-05 Missing parameter in `RewardTokenSupplyIndexUpdated` event

There's two similar events, `RewardTokenSupplyIndexUpdated` and `RewardTokenBorrowIndexUpdated`, the first one do not notify the updated value, while the second one does.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Rewards/RewardsDistributor.sol#L89-L93

```solidity
File: contracts\Rewards\RewardsDistributor.sol
89:     /// @notice Emitted when a reward token supply index is updated
90:     event RewardTokenSupplyIndexUpdated(address vToken);

91:
92:     /// @notice Emitted when a reward token borrow index is updated
93:     event RewardTokenBorrowIndexUpdated(address vToken, Exp marketBorrowIndex);

```

## N-06 Missing Natspec comment for `initializeMarket` function

RewardDistributor::initializeMarket function has no Natspec comment while this is an important function.

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Rewards/RewardsDistributor.sol#L125