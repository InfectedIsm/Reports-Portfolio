______________________________
# 🔴 H01 - `DrawManager::finishDraw` can revert when the sum of rewards to distribute is more than available reserve 

## Summary
For each draw, depending on how the auction is concluded, the drawers will receive a fraction of the `reserve`.
If the sum of all fractions to distribute is greater than 100%, then `finishDraw`, which is responsible for calling the `PrizePool::awardDraw` will revert, preventing the draw to be awarded with no possibility for the function to recover until next draw.

## Vulnerability Detail
To be awarded, the draw needs two things to happen: 
1) a random number must be generated: this is achieved [by calling `RngWitnet::startDraw()`](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-rng-witnet/src/RngWitnet.sol#L132-L135) which itself calls [`DrawManager::startDraw()`](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L219-L219). This step take some time as the random number (RN) generation is executed by the Witnet blockchain, which return the value to the requesting chain after few minutes. This function cannot be called until next draw starts, or RNG fails.
2) once the RN is returned by the oracle, `finishDraw` must be called by a drawer to send it to the Prize Pool so that prizes can be awarded. This is during that call that all reward are computed and awarded to drawers.

The protocol incentivize drawers by running a [PFDA auction](https://dev.pooltogether.com/protocol/design/draw-auction), where the reward allocated for doing one of these 2 actions increase over time.
The actual implemented PFDA algorithm do not return a price, but returns **a fraction of the total `reserve`** that will be attributed to the drawer.

As we said in (1), if the RN generation fails, anyone can call again `startDraw()` to run a new RNG. Once a valid RN is returned, all drawers that participated in the generation (failed generation and successful) will receive the expected reward, as re-running the function will [add the drawer to an array](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L253-L258) to be [processed afterward for reward distribution](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L349-L351). The maximum number of try [is capped](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L105) by `maxRetries`.
So, there can be up to `maxRetries` startDrawRewards, and one finishDrawReward.

The issue is that the computation of the fraction allocated to each reward is done independentely to the other already planned  reward fractions, and computed solely on the timing of the auction.

We can infer that when (a) startDraw auctions are done more than once and (b) when auctions are completed "late", the sum of the returned fractions by the PFDA can be greater than 1.
If this happens, [when reward will be distributed through `_reward()`](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L349-L352), which calls `PrizePool::allocateReserveFromReserve` [will revert](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-prize-pool/src/PrizePool.sol#L438-L440) for the reward that try to allocate more than the available `_reserve` (as previous allocated rewards reduced it)

## Impact
The draw cannot be completed and will be discarded, thus disrupting the awarding process and prize distribution.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L105
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-draw-manager/src/DrawManager.sol#L349-L352
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-prize-pool/src/PrizePool.sol#L438-L440

## Tool used
Manual review

## Recommendation
Don't allow rewards to be greater than available reserve. 
This can be done either by saving the remaining reserve each time a startDraw has been conducted and reward calculated, or by checking that the sum of the fraction is less than 1, and if not the case, normalizing the fractions such as the sum is <=1.
_______________________________________



# 🟡 M01 - ClaimPrize hooks gas limit can be violated using a gas bomb, making `claimPrizes` revert

## Summary
`beforeClaimPrize` and `afterClaimPrize` hooks calls execution receive a finite amount of gas dicated by the state variable `HOOK_GAS` which is set to 150k.
In this report we will show that it is possible for a hook to bypass that limit and consume more gas than `HOOK_GAS`, which can lead in worst case, to not letting enough gas to the claimer to complete its array of claims.
The attack is based on the [unbounded return data](https://github.com/kadenzipfel/smart-contract-vulnerabilities/blob/master/vulnerabilities/unbounded-return-data.md) attack vector, also [explained here](https://twitter.com/0xkarmacoma/status/1763746082537017725) which combine the use of assembly and returned value (or revert message) to force caller to expand memory.
We will then provide a simple remediation against it.

## Vulnerability Detail
Claimers are users/bots incentivized to claim prize on behalf of winners. They get a portion of the prize value as a reward for doing it. 
Claimers [can claim multiple prizes](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L93) at once using the `claimPrizes` function from the `Claimer` contract.
These prizes are then called [each one after the other](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L158-L169) in a loop inside a try/catch structure.
If a fail occur during a claim, the try/catch simply emit an event, and claim the next prize.

Winners are allowed to set-up two hooks that will be executed before and after the prize claim, each one receiving HOOK_GAS amount of gas to spend for the call. This allow claimers to know how much gas will be spent for the call to be executed and configure in an optimal way their call.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-vault/src/abstract/Claimable.sol#L76-L121
```solidity
File: pt-v5-vault/src/abstract/Claimable.sol
076:     function claimPrize(
...:           /* claimPrize parameters */
082:     ) external onlyClaimer returns (uint256) {
083:         address _prizeRecipient;
084:         bytes memory _hookData;
085: 
086:         if (_hooks[_winner].useBeforeClaimPrize) {
087:             (_prizeRecipient, _hookData) = _hooks[_winner].implementation.beforeClaimPrize{ gas: HOOK_GAS }(
...:           /* beforeClaimPrize parameters */
093:             );
...:
...:           /* some code ... */
...:
100:         uint256 _prizeTotal = prizePool.claimPrize(        
...:           /* claimPrize parameters */

107:         );
108: 
109:         if (_hooks[_winner].useAfterClaimPrize) {
110:             _hooks[_winner].implementation.afterClaimPrize{ gas: HOOK_GAS }(
...:           /* afterClaimPrize parameters */
117:             );
118:         }
119: 
120:         return _prizeTotal;
121:     }
```

But using a gas bomb function as the one shared in the `Summary` section, it is possible to craft hooks that will spend more gas than the configured limit.
If a hook consume almost all available gas for the complete execution of the entire claim array, claimers might see their execution run out of gas, preventing them to finish execute the claiming loop, which can unexpectedely make the whole claimPrizes call revert, which means claimer will have claimed no prize.

## Impact
By disrupting the claiming process and making the whole `claimPrizes` call revert, the claimer will not earn any reward.

The cost of that attack is low, as it simply require the attacker to deploy a gas bomb contract once and set it as a hook.

To increase the anoyance of the attack, the attacker could target a specific pool, participating to a prize vault with multiple accounts, all of them having that malicious prize hook pointing to the same gas bomb contract, increasing even more the gas bomb effect if multiple of these account win prizes (which can be pretty frequent for lower tiers [as shown in the dashboard](https://analytics.cabana.fi/optimism))
The funds used to fund those accounts are not lost, and can make him earn prizes. He can even address-check the hook call, and "desactivate" the gas bomb for his bot.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-vault/src/abstract/Claimable.sol#L76-L121
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L93
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L158-L169

## Tool used
Manual review

## Recommendation
Use the [ExcessivelySafeCall](https://github.com/nomad-xyz/ExcessivelySafeCall) library or the [LowLevelHelpers](https://github.com/ProjectOpenSea/seaport-core/blob/d4e8c74adc472b311ab64b5c9f9757b5bba57a15/src/lib/LowLevelHelpers.sol#L26) method used [by Seaport](https://twitter.com/z0age/status/1763936575862522114)

_______________________________________
# 🟡 M02 - `Claimers` can receive less `feePerClaim` than they should if some prizes are already claimed or if reverts because of a reverting hook

## Summary

If a claimer propose an array of prizes to claim, but some of these prizes have already been claimed, or some claims revert, then the actual `feePerClaim` received will be less **compared to what it should really be, as expected by the VRGDA algorithm**

This happens because `_computeFeePerClaim` can undervaluate the value of `feePerClaim`, as it compute failed claims as successful ones.

This poses an issue, as `feePerClaim` is then used as an input by `_vault.claimPrize(_winners[w], _tier, _prizeIndices[w][p], _feePerClaim, _feeRecipient)`, which will transfer the undervaluated fee to the claimer.


## Vulnerability Detail
Auctions for claiming prizes are based on the [VRGDA algorithm](https://www.paradigm.xyz/2022/08/vrgda)
In simple terms, this algorithm update price depending on either the numbers of claim is behind or ahead of time schedule.
In order to have a schedule, a target of claim per time unit is defined.
Just to give an idea, let's simplify that to the extreme (we will see complete formula afterward) and say : `price(t) = claim/expected * targetPrice(t)` 
E.g: if 10 claims per 2 hours are exepected, then at t=1h, 5 claims should be concluded.
If only 4 were claimed, then we can calculate that (4 claims)/(5 expected) < 1, price will be lower that target.
if 6 were claimed, then we will have (6 claims)/(5 expected) > 1, price will be greater than target.

The formula that has been implemented into `LinearVRGDALib` is the following:

```
price = p0 * e^ (k * (t - n+1/r))   ; with k = ln(maxFee/minFee) * t_target
```

With:
- `n` the number of claim already completed
- `r` the expected rate per hour
- `k` the decay constant (speed at which price will change)
- `p0` the target price (or fee in our case)

The more `k *(t - n+1/r) > 0`, the more `price > p0`
When `t = n+1/r` <=> `(k * (t - n+1/r)) = 0`, then `price = p0`
The more `k * (t - n+1/r) < 0`, the more `price < p0`

We understand that the more whe are behind schedule in term of expected claim, the higher the fees earned by claimer will be.
And the more we are ahead of schedule, the lower the fee for claimers will be (as their is no urgency)

### Scenario
Now let's see what happens in this scenario:
1. For now, 0 prizes have already been claimed from the prize pool, `n = 0`
2. Alice has 5 prizes to claim, she build her claim array
3. It seems that 2 of the prizes Alice was going to claim are claimed right before, now `n = 2`
4. Alice tx is executed with the 5 prizes

Now let's see what happen from a code perspective.
The code above is the entry point for claiming prizes. As we can see L113, the `feePerClaim` is computed based on the [number of claims to count](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L126-L136) (`_countClaims`) and the number of [already claimed prizes](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-prize-pool/src/PrizePool.sol#L302-L302) (`prizePool::claimCount()`)
Then, the computed `feePerClaim` value is given to `_claim` which actually claim the prizes into the Prize Pool.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
```solidity
File: pt-v5-claimer/src/Claimer.sol

090:   function claimPrizes(
091:     IClaimable _vault,
092:     uint8 _tier,
093:     address[] calldata _winners,
094:     uint32[][] calldata _prizeIndices,
095:     address _feeRecipient,
096:     uint256 _minFeePerClaim
097:   ) external returns (uint256 totalFees) {
...:
...:          /* some code */
...:
112:     if (!feeRecipientZeroAddress) {
113:       feePerClaim = SafeCast.toUint96(_computeFeePerClaim(_tier, _countClaims(_winners, _prizeIndices), prizePool.claimCount()));
114:       if (feePerClaim < _minFeePerClaim) {
115:         revert VrgdaClaimFeeBelowMin(_minFeePerClaim, feePerClaim);
116:       }
117:     }
118:
119:     return feePerClaim * _claim(_vault, _tier, _winners, _prizeIndices, _feeRecipient, feePerClaim); 
120:   }
```

Now, let's see how `_computeFeePerClaim` actually compute `feePerClaim`.
We see above L230-241 that a fee is calculated for each of the claims of the array, starting at `_claimedCount` (The number of prizes already claimed) based on the VRGDA formula L309. The returned value (which is stored into `feePerClaim`) is the averaged fee as shown L241.
And as we explained earlier, the higher the number of claim we make, the lower the earned fee are. So, a higher value of `_claimedCount + i` will give lower fees.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236
```solidity
File: pt-v5-claimer/src/Claimer.sol
205:   /// @param _claimCount The number of claims to check
206:   /// @param _claimedCount The number of prizes already claimed
207:   /// @return The total fees for the claims
208:   function _computeFeePerClaim(
209:     uint8 _tier,
210:     uint256 _claimCount,
211:     uint256 _claimedCount
212:   ) internal view returns (uint256) {
...:
...:        /* some code */
...:
227:     uint256 elapsed = block.timestamp - (prizePool.lastAwardedDrawAwardedAt());
228:     uint256 fee;
229: 
230:     for (uint256 i = 0; i < _claimCount; i++) {
231:       fee += _computeFeeForNextClaim(
232:         targetFee,
233:         decayConstant,
234:         perTimeUnit,
235:         elapsed,
236:         _claimedCount + i,         
237:         _maxFee
238:       );
239:     }
240: 
241:     return fee / _claimCount;
242:   }
...:
...:        /* some code */
...:
301:   function _computeFeeForNextClaim(
302:     uint256 _targetFee,
303:     SD59x18 _decayConstant,
304:     SD59x18 _perTimeUnit,
305:     uint256 _elapsed,
306:     uint256 _sold,
307:     uint256 _maxFee
308:   ) internal pure returns (uint256) {
309:     uint256 fee = LinearVRGDALib.getVRGDAPrice(
310:       _targetFee,
311:       _elapsed,
312:       _sold,
313:       _perTimeUnit,
314:       _decayConstant
315:     );
316:     return fee > _maxFee ? _maxFee : fee;
317:   }
318: 
```

What we can see from this, is that the computation will be executed for `_claimCount = 2` and up to `i = 5`, so as if there has been 7 claimed prizes, while in reality only 5 prizes are claimed, leading in an undervaluation of the fees to receive.
As you probably have infered, the computation should have been made for `i = 3` to be correct.

## Impact
The `feePerClaim` computation is incorrect as the VRGDA is calculated for more claims that will really happen, leading to less fee earned by claimers at the time of the call.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236

## Tool used
Manual Review

## Recommendation
The `PrizePool` contract expose a function to check if a prize has already been claimed: `wasClaimed`
This can be used to countClaims based on the actual true number of claimable prizes from the array.

This isn't a "perfect" solution though, as there are still issues when not already claimed prizes revert because of reverting prize hooks. In that case, VRGDA will still count the claim as happening, but we can consider this less likely to happen.

```diff
  function _countClaims(
    address[] calldata _winners,
    uint32[][] calldata _prizeIndices
  ) internal pure returns (uint256) {
    uint256 claimCount;
    uint256 length = _winners.length;
    for (uint256 i = 0; i < length; i++) {
-     claimCount += _prizeIndices[i].length;
+	  numPrize = _prizeIndices[i].length;
+	  for(uint256 j = 0; j < numPrize; j++) {
+     	bool wasClaimed = wasClaimed(_vault, _winner, _drawId,_tier, _prizeIndex);
+     	if(!wasClaimed) {
+		 claimCount += 1;
+		}
+     }
    }
    return claimCount;
  }
```
