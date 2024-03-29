# Underflow in `AbstractStakingAM._getRewardBalances` will cause a DoS of all operations during a period of time

## Summary
`deltaReward` in `AbstractStakingAM._getRewardBalances` underflow each time reward are collected, and will stop to underflow once enough rewards have accrued

## Vulnerability Detail
On Stargate Protocol, users can stake in available pools to receive LP tokens. 
Then, they can stake these LP tokens in the Stargate LPStakingTime contract to get more rewards.

`StakedStargetAM` allow users to stake their Stargate LP tokens inside Arcadia. 
To stake, users must first `mint` what is called a `position`, which is an ERC721 representing their deposit inside the contract.
This position keep tracks of how much they staked, and how much reward they are entilted to.
They can then update this position by calling `burn` (which will withdraw all stake and delete the position and burn its ERC721), `increaseLiquidity` or `decreaseLiquidity`.
So `StakedStargetAM` stake for the users, and redistribute the rewards to each user based on how much they participated in the total stake.
Most of this calculation is done inside the `_getRewardBalances` function, which update the states of the positions and some other global states.
`_getRewardBalances` is called on each interaction with the pool by a user, which happens in `mint`, `burn`, `increaseLiquidity` and `decreaseLiquidity`
The issue reside inside `_getRewardBalances`, where an underflow can occur when two calls from any of these functions happen in a certain timeframe.

Now let's see how this happens.
https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L529-L548
```solidity
    function _getRewardBalances(AssetState memory assetState_, PositionState memory positionState_)
        internal
        view
        returns (AssetState memory, PositionState memory)
    {
        if (assetState_.totalStaked > 0) {
            // Calculate the new assetState
            // Fetch the current reward balance from the staking contract.
            uint256 currentRewardGlobal = _getCurrentReward(positionState_.asset);
            // Calculate the increase in rewards since last Asset interaction.
			uint256 deltaReward = currentRewardGlobal - assetState_.lastRewardGlobal; //<@ underflow happens here
            uint256 deltaRewardPerToken = deltaReward.mulDivDown(1e18, assetState_.totalStaked);
            // Calculate and update the new RewardPerToken of the asset.
            // unchecked: RewardPerToken can overflow, what matters is the delta in RewardPerToken between two interactions.
            unchecked {
                assetState_.lastRewardPerTokenGlobal =
                    assetState_.lastRewardPerTokenGlobal + SafeCastLib.safeCastTo128(deltaRewardPerToken);
            }
            // Update the reward balance of the asset.
            assetState_.lastRewardGlobal = SafeCastLib.safeCastTo128(currentRewardGlobal);
```

`currentRewardGlobal`, which calls `_getCurrentReward`, is in fact calling the `LpStakingTime` Stargate contract :
```solidity
    function _getCurrentReward(address asset) internal view override returns (uint256 currentReward) {
        currentReward = LP_STAKING_TIME.pendingEmissionToken(assetToPid[asset], address(this));
    }
```

This function return the pending rewards since last reward collection. So when reward have been collected, the value starts back from 0 and increase over time
On Stargate `LPStakingTime` contract, reward collections happens everytime a user deposit or withdraw from the pool. ([LpStakingTime verified contract on Base](https://basescan.deth.net/address/0x06eb48763f117c7be887296cdcdfad2e4092739c))
Now, if we go back to Arcadia `AbstractStakingAM`, this happens everytime `mint`, `burn`, `increaseLiquidity` and `decreaseLiquidity` is called by a user.
So let's say a user calls `increaseLiquidity`.
First `increaseLiquidity`, `_getRewardBalances` [is called](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L340). 
So let's say there are `1000` pending rewards, this means `currentRewardGlobal` will be equal to 1000.
This value will then be saved to `assetState_.lastRewardGlobal`, function return to `increaseLiquidity`, which will stake the user deposit into Stargate ([here](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L314) and [here](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol#L82-L87)) and trigger the reward collection.

Now, what happens if `_getRewardBalances` is called again ? (by a user interacting again)
Answer: the pending rewards starts back to zero (and increment with time going on) 
This means, during a period of time, `deltaReward = currentRewardGlobal - assetState_.lastRewardGlobal` **will underflow**
And the period of time where the underflow occur is roughly equal to the time between two successful reward collection (modulated by a factor of the quantity of token staked during this period, as more/less stake will increase/decrease reward accrual in Stargate).

This means depending on how often the `StakedStargetAM` is interacted, the DoS will be more or less lengthy.
If it happens that the pool is not used for few hours while there's a lot of token staked inside, the underflow will persist until these few hours finishes.

## Impact
Intempestive denial of service of the contract by user not even acting maliciously.
But we could also imagine a malicious user front-running legitimate interaction, which require only 1 wei of deposit/withdrawal per interaction.

## Code Snippet
Please refer to this gist for the PoC : https://gist.github.com/InfectedIsm/247c12f6234f296ff5975d9881cd74af

## Tool used
Manual Review

## Recommendation
The `_getRewardBalance` should be refactored to not allow `deltaReward` to underflow.
The issue comes from the fact that once reward are collected, `currentRewardGlobal` will be smaller than `assetState_.lastRewardGlobal` until enough reward is accrued.
Adding a condition for `currentRewardGlobal < assetState_.lastRewardGlobal` is necessary and a first step, but for a proper solution, some thinking should be taken to keep the reward distribution correct.
I'm not sure, but maybe devs thought that `LP_STAKING_TIME.pendingEmissionToken` would return an ever increasing value. In this case, I think that what would fit should be to simply have `uint256 deltaReward = currentRewardGlobal`