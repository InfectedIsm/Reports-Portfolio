# [M-01] - A malicious prize winner can steal the claimReward from the claimer using the BeforeClaimPrize hook

## Vulnerability details
Pool Together prize distribution is based on incentivization of 3rd parties to activate its functionality, rather than depending on a governance or centralized actor. For example in this case, everytime a draw to award prizes is closed, any 3rd party can call the `claimable::claimPrize` function to claim the winner's prize on his behalf. By doing so, the prize will be distributed to the winner, minus a small fee rewarded to the claimer.

As said above, to claim prizes 3rd party must call the `Claimable::claimPrize`:

https://github.com/code-423n4/2024-03-pooltogether/blob/480d58b9e8611c13587f28811864aea138a0021a/pt-v5-vault/src/abstract/Claimable.sol#L76-L119

```solidity
File: pt-v5-vault\src\abstract\Claimable.sol
076:     function claimPrize(
077:         address _winner,
078:         uint8 _tier,
079:         uint32 _prizeIndex,
080:         uint96 _reward,
081:         address _rewardRecipient
082:     ) external onlyClaimer returns (uint256) {
083:         address recipient;
084: 
085:         if (_hooks[_winner].useBeforeClaimPrize) {
086:             recipient = _hooks[_winner].implementation.beforeClaimPrize{ gas: HOOK_GAS }(
087:                 _winner,
088:                 _tier,
089:                 _prizeIndex,
090:                 _reward,
091:                 _rewardRecipient
092:             );
093:         } else {
094:             recipient = _winner;
095:         }
096: 
097:         if (recipient == address(0)) revert ClaimRecipientZeroAddress();
098: 
099:         uint256 prizeTotal = prizePool.claimPrize(
100:             _winner,
101:             _tier,
102:             _prizeIndex,
103:             recipient,
104:             _reward,
105:             _rewardRecipient
106:         );
```

And as we can see here, before the prize is claimed (L99), a hook is called (L86).
We can observe that this hook receives the exact same parameters as the `claimPrize` call: `_winner`, `_tier`, `_prizeIndex`, `reward` and `rewardRecipient`.

The vulnerability lies in the fact that nothing prevent a malicious winner to configure a hook that will claim the prize before the original claimer.
By developping a hook that reuses these parameters to call `claimable::claimPrize`, but replacing the `_rewardRecipient` address with one of its own, the winner can then receive its legitimate prize + the claimer reward.
And as a bonus, we can also note that the winner will not even have to pay the gas fee for the call, as it uses the claimer call to execute this action.

## Impact
Claimer will lose part of their profit. 
If too many winners set such a hook, this can disencentivize claimers to do their job.

## Proof of Concept

By claiming right before the original claimer thanks to the hook, the winner sets the `_claimedPrizes[msg.sender][_winner][lastAwardedDrawId_][_tier][_prizeIndex])` entry to `true` (L516), after that the award will be credited to him (L525)
Then, when the claimer will enter `PrizePool::claimPrize` himself after the hook has been executed, the entry will already be set to `true`, making if statement at L512 pass and the call return early.

https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4380f1ab65e39109ac6a36f8935137a2ef685860/src/PrizePool.sol#L512-L516
```solidity
File: pt-v5-vault\lib\pt-v5-prize-pool\src\PrizePool.sol
476:   function claimPrize(
477:     address _winner,
478:     uint8 _tier,
479:     uint32 _prizeIndex,
480:     address _prizeRecipient,
481:     uint96 _claimReward,
482:     address _claimRewardRecipient
483:   ) external returns (uint256) {
484:     
...: /* removed some code for readability */
511: 
512:     if (_claimedPrizes[msg.sender][_winner][lastAwardedDrawId_][_tier][_prizeIndex]) { 
513:       return 0; //<@ original claimer will return here because of the hook that set it first to true L516
514:     }
515: 
516:     _claimedPrizes[msg.sender][_winner][lastAwardedDrawId_][_tier][_prizeIndex] = true;
517: 
518:     // `amount` is a snapshot of the reserve before consuming liquidity
519:     _consumeLiquidity(tierLiquidity, _tier, tierLiquidity.prizeSize);
520: 
521:     // `amount` is now the payout amount
522:     uint256 amount;
523:     if (_claimReward != 0) {
524:       emit IncreaseClaimRewards(_claimRewardRecipient, _claimReward);
525:       _rewards[_claimRewardRecipient] += _claimReward;
526: 
527:       unchecked {
528:         amount = tierLiquidity.prizeSize - _claimReward;
529:       }
530:     } else {
531:       amount = tierLiquidity.prizeSize;
532:     }
533: 
534:     // co-locate to save gas
535:     claimCount++;
536:     _totalWithdrawn = SafeCast.toUint128(_totalWithdrawn + amount);
537:     _totalRewardsToBeClaimed = SafeCast.toUint104(_totalRewardsToBeClaimed + _claimReward);
538: 
...: /* removed the emited event */
550: 
551:     prizeToken.safeTransfer(_prizeRecipient, amount);
552: 
553:     return tierLiquidity.prizeSize;
554:   }
```
## Tools Used
Manual review

## Recommended Mitigation Steps
Hopefully, the solution is pretty easy to set-up: just add a `nonReentrant` modifier (can use the ReentrancyGuard lib from OZ) to the `Claimable::claimPrize`
This way, the hook will not be allowed to reenter to function.


[M-02] Nothing prevent from withdrawing the yield buffer




__________________________________________________________

[L-01] Inconsistencies with IERC4626

from https://eips.ethereum.org/EIPS/eip-4626:

- `totalAsset` MUST be inclusive of any fees that are charged against assets in the Vault.
The fees are not accounted for as we can see here, they are comprised in the totalAssets while they should me substracted.
```solidity
File: pt-v5-vault\src\PrizeVault.sol
334:     /// @inheritdoc IERC4626
335:     /// @dev The latent asset balance is included in the total asset count to account for the "dust collection 
336:     /// strategy".
337:     function totalAssets() public view returns (uint256) {
338:         return yieldVault.convertToAssets(yieldVault.balanceOf(address(this))) + _asset.balanceOf(address(this)); 
339:     }	
340: 
```


[L-02] `PrizeVaultFactory::deployVault` uses an immutable variable to configure the yieldBuffer

As stated by the comment, the yield buffer should be configurable depending on the decimals, and the dollar value of the asset of the vault. This is not a big issue here as a `PrizVault` can also be deployed not using the factory if it needs a different yield buffer, but this means it will not be added to the `allVaults` array to be easily found by 3rd parties working with Pool Together.

It is probably a better idea to let the user configure it correctly, maybe still forcing a minimum value of 1e5 for it to be effetive.

```solidity
File: pt-v5-vault\src\PrizeVaultFactory.sol
059:     /// If the yield buffer is depleted on a vault, the vault will prevent any further 
060:     /// deposits if it would result in a rounding error and any rounding errors incurred by withdrawals
061:     /// will not be covered by yield. The yield buffer will be replenished automatically as yield accrues
062:     /// on deposits.
063:     uint256 public constant YIELD_BUFFER = 1e5;
...: /* removed unnecessary lines */
092:     function deployVault(
093:       string memory _name,
094:       string memory _symbol,
095:       IERC4626 _yieldVault,
096:       PrizePool _prizePool,
097:       address _claimer,
098:       address _yieldFeeRecipient,
099:       uint32 _yieldFeePercentage,
100:       address _owner
101:     ) external returns (PrizeVault) {
102:         PrizeVault _vault = new PrizeVault{
103:             salt: keccak256(abi.encode(msg.sender, deployerNonces[msg.sender]++)) //@audit vulnerable to front-running
104:         }(
105:             _name,
106:             _symbol,
107:             _yieldVault,
108:             _prizePool,
109:             _claimer,
110:             _yieldFeeRecipient,
111:             _yieldFeePercentage,
112:             YIELD_BUFFER, 				//<@ should be configurable
113:             _owner
114:         );
...:
120:         allVaults.push(_vault);
121:         deployedVaults[address(_vault)] = true;
122:
```

[L-03] If the Yield Vault is Vulnerable to an Inflation Attack, funds can stay in Prize Vault on first depositor rather than being deposited in the Yield Vault

The PrizeVault should check that minted shares are non zero, in case of a vulnerable Yield Vault.
Otherwise, a unprotected Yield Vault would allow an attacker to inflate the share price . E.g: Solmate do not implement the decimal offset by default.
Another workaround would be to mint dead shares to the yield vault in PrizeVaultFactory::deployVault when deploying the PrizeVault if its a new one.

The steps are:
1) Alice deposit 1000 assets into the PrizeVault 
2) Bob front-run Victim by depositing 1 wei in YieldVault, then donating 1000 assets to YieldVault too
3) Alice tx is executed, 1000 assets are deposited into PrizeVault but not deposited to the Yield Vault.
4) Bob can take back its assets in the Yield Vault

In this state, the impact is really low as Alice can still get here asset backs from the Prize Vault by buring here shares, though this could potentially be escalated to something bigger.