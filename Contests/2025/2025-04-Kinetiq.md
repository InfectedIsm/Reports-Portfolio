
# Kinetiq - Findings Report

# Table of contents
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    -  [H-01. `processValidatorRedelegation` cannot work in the current state](#H-01)
- ## Medium Risk Findings
    -  [M-01. Processing withdrawals before deposits can cause some deposit to not be delegated](#M-01)
    -  [M-02. `redelegateWithdrawnHYPE` cannot work in the current state](#M-03)
- ## Low Risk Findings
    -  [L-01. Missing slippage protection on `queueWithdrawal`](#L-01)
- ## Accepted Risk (Not Rewarded)
    -  [M-04. `_processL1Withdrawals` should call `L1Read` to ensure correct execution of `L1Write` calls, otherwise they will fail silently](#M-04)


# HIGH

## 🔴 H-01 - `processValidatorRedelegation` cannot work in the current state

### Summary
`StakingManager::processValidatorRedelegation` is called through `ValidatorManager::closeRebalanceRequests`. This is the next action that must be done after `ValidatorManager::rebalanceWithdrawal` has been called and processed. `rebalanceWithdrawal` undelegate from previous validators, while `processValidatorRedelegation` re-delegate the the undelegated balance to the new operator.

The issue is that the function confuse the EVM balance and the different HyperCore L1 balances, failing to redelegate the tokens. 
### Vulnerability details

Before diving into the flow of the issue, it is important to recall the different balances that exists in Hyperliquid:
- E**VM balance**: this is the balance of the asset on the HyperEVM, if its the native token it can be read with `address(...).balance` as we would do on EVM for ETH.
- **L1 Spot balance**: this balance lives outside of the HyperEVM, on the HyperCore, and is [transferred that way](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers)
- **L1 [Staking](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/staking) balance**: it is an intermediary balance, from where assets can then be delegated to validators. Users can move tokens from "spot" to "staking" (or the opposite direction) using the [`L1Write` contract](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore#write-system-contract-testnet-only)
- **L1 Delegated balance**: this balance is a subset of the staking balance, only "staking" balance can be delegated. Undelegating assets will move then from "delegated" back to "staking".

Before executing its logic, the [`StakingManager::processValidatorRedelegation`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L362-L368) check the HYPE native balance `L365`, and reverts if the balance is less than the amount to redelegate:

```solidity
File: src/StakingManager.sol
362:     function  processValidatorRedelegation(uint256 amount) external nonReentrant whenNotPaused {
363:         require(msg.sender == address(validatorManager), "Only ValidatorManager");
364:         require(amount > 0, "Invalid amount");
365:         require(address(this).balance >= amount, "Insufficient balance");
366: 
367:         _distributeStake(amount, OperationType.RebalanceDeposit);
368:     }
369: 
```

But this balance shouldn't be checked, as the tokens are never withdrawn from the staking balance in the process of `ValidatorManager::rebalanceWithdrawal` that is called before. Let's see how.  

When `ValidatorManager::rebalanceWithdrawal` is called, a new `validatorRebalanceRequests` entry [is created](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/ValidatorManager.sol#L234-L234) storing the amount to rebalance. The function then [calls `SM::processValidatorWithdrawals`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/ValidatorManager.sol#L219-L219), which itself [calls `_withdrawFromValidator`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L350-L350) with `OperationType.RebalanceWithdrawal`, creating a new `_pendingWithdrawal` entry to be processed later.  

When the pending withdrawal is finally processed through [`_processL1Withdrawals`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L689-L701), the only balance change happening here  is the undelegation of the tokens, [as `operationType == RebalanceWithdrawal`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L696-L698), which do not satisfy the if case.  

This means that the tokens are still on the "staking" balance and not moved to "spot" once the withdrawal processed.  

Now, if we go back the to `processValidatorRedelegation`, we understand why the `require(address(this).balance >= amount, "Insufficient balance")` statement can only revert: never tokens have been moved back to EVM.  
Even the operator function [`withdrawFromSpot`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L994-L997) cannot help with the revert here, as the function moves tokens from "spot" to "EVM", but as we've seen above, no tokens were moved to "spot" yet.
### Impact
Impact is high, as `processValidatorRedelegation` cannot work properly.
A collateral damage is that if `closeRebalanceRequests` always fail, this means that the validator will never be removed from `_validatorsWithPendingRebalance`,with the effect that the `OracleManager` [cannot generate performance for a validator with a pending rebalance](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/OracleManager.sol#L151-L151).
Likelihood is high too, as rebalance are an important part of the process of the protocol as shown by its capabilities.
### Recommended Mitigation Steps

As the `rebalanceWithdrawal` function do not unstake tokens back to "spot" and keep them in the "staking" balance, the latter value should be used as a requirement before calling `processValidatorRedelegation` rather than `address(this).balance`.
The staked and undelegated balance can be check by calling `L1Read::delegatorSummary` which returns the [`undelegated` balance](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/lib/L1Read.sol#L31-L36).


 

# MED

## 🟡 M-01 -Processing withdrawals before deposits can cause some deposit to not be delegated

### Summary
The way withdrawals and deposits are processed in `processL1Operations` (all withdrawals requests first, then all deposits) can lead in some cases, to balance not being delegated, which ultimately reduce earned rewards from validator, making the vault less profitable than expected.
### Vulnerability details

Kinetiq allow users to deposit HYPE (native currency of Hyperliquid) into the `StakingManager`, and receive kHYPE (a share of the vault) in exchange. The HYPE tokens are then sent to the L1 by the `StakingManager` in order to delegate the tokens to a validator and earn rewards, redistributed to depositor through it shares.

Before diving into the flow of the issue, there is two mechanisms are important to understand.  
First, the different balances that exists in Hyperliquid:
- E**VM balance**: this is the balance of the asset on the HyperEVM, if its the native token it can be read with `address(...).balance` as we would do on EVM for ETH.
- **L1 Spot balance**: this balance lives outside of the HyperEVM, on the HyperCore, and is [transferred that way](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/hypercore-less-than-greater-than-hyperevm-transfers)
- **L1 [Staking](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/staking) balance**: it is an intermediary balance, from where assets can then be delegated to validators. Users can move tokens from "spot" to "staking" (or the opposite direction) using the [`L1Write` contract](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/interacting-with-hypercore#write-system-contract-testnet-only)
- **L1 Delegated balance**: this balance is a subset of the staking balance, only "staking" balance can be delegated. Undelegating assets will move then from "delegated" back to "staking".

Second, we must understand which functions are taking part in the process, and how these different balances are affected (in the following part, the `StakingManager` contract will be referred as `SM`):
1. [`stake()`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L219-L252)– *called by user to deposit HYPE*
    1. `user` deposits HYPE to `SM`
    2. `SM` move HYPE from EVM to "spot" ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L476-L476))
    3. `SM` move balance from “spot” to “staking”  ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L481-L481))
    4. `SM` create a `_pendingDeposit` entry ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L616))
2. [`queueWithdrawal()`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L256-L296)– *called by user to request a withdrawal from validators*
    1. `SM` create a `_pendingWithdrawal` entry ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L614))
3. [`processL1Operations()`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L627-L666)–  *called by operator to process withdrawals and deposits*.
4. [`_processL1Withdrawals()`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L673-L714) – *called by `processL1Operations`*
    2. `SM` undelegate "staking" (subject to 1 day delegate delay)  ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L693))
    3. `SM` move "staking" to "spot" (subject to 7 days delay)  ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L697))
5. [`_processDeposits()`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L721-L757)– *called by `processL1Operations`*
    1. `SM` delegate "staking" (every time the function is called, undelegate cannot be called for 1 day) ([code](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L741))

Now, we can start to describe the following scenario *(we'll set an  exchange rate of 1:1 for HYPE/KHYPE, and 10% withdrawal fees for simplicity):*
1. Alice stake 10 HYPE (and receive 10 kHYPE)
2. Bob stake 1 HYPE (and receive 1 kHYPE)
3. Carol stake 1 HYPE (and receive 1 kHYPE)
4. Alice queue withdrawal for 10 kHYPE (which result in 9 HYPE to be withdrawn after fees)
5. Bob queue withdrawal for 1 kHYPE (which result in 0.9 HYPE to be withdrawn)

After these operations, the `processL1Operation` will be called by an operator in order to update the balances and ensure assets are delegated and earning rewards.  
 Throughout the call, elements will be [processed in the order they were added](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L688-L689) in their respective array, and [withdrawals first, then deposits](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L641-L647).  
The processing happens in `processL1Operations` which itself calls `_processL1Withdrawals` and `_processL1Deposits`
The delegation to operator [happens in deposits](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L741-L741), while the un-delegation from operator, and withdrawal from "staking" to "spot" balance [happens in withdrawal](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L692-L698).  
This means that in the above example, things will happen that way:
1. **Alice's withdrawal is processed first:** undelegation fails (as nothing has been delegated yet), withdrawal 9 HYPE from "staking" to "spot" succeed (reduce the staking balance available to delegate)
2. **Bob's withdrawal is processed second:** undelegation fails (same reason), withdrawal of 0.9 HYPE from "staking" to "spot" succeed, now equal to 9.9 HYPE being unstaked.
3. **Alice's deposit is processed**, which tries to delegate 9 HYPE, but as they are already in the withdrawal process, this fails as there isn't enough "staking" balance to delegate.
4. **Bob's deposit is processed** for 1 HYPE and successfully pass, making the delegated balance equal to 1 HYPE.
5. 4. **Carol's deposit is processed** for 1 HYPE and successfully pass, making the delegated balance equal to 2 HYPE.

So, at the end of the whole process we will have this state:
- L1 Spot: `0 HYPE` (still in unstaking queue for 7 days)
- L1 "Unstaking Queue": `9.9 HYPE`
- L1 Staking: `0.1 HYPE`
- L1 Delegated: `2 HYPE`

But now, let's see what should have been the balance if everything went correctly.
1. Alice deposited 10 and withdrawn 9, so 1 HYPE should  be delegated
2. Bob deposited 1 and withdrawn 0.9, so a total of 1.1 HYPE should be delegated
3. Carol deposited 1, so **a total of 2.1 HYPE should be delegated**

We now see that 0.1 HYPE are missing in delegation.  
This discrepancy in delegated balance will reduce the vault profitability as it will earn less rewards than expected.  
### Impact
Discrepancy in L1 balances management, causing some amount to not be delegated, thus reducing profitability of the vault from what is expected.

### Recommended Mitigation Steps
Care must be taken for the implementation of a fix here, as the below "solution" only work for operationType related to user deposits and withdrawals, and specific process might be necessary for other operationType.  

In a loop, adds up all withdrawal and deposit request amount to be processed, and only process the needed amount.
E.g, `int256 amountToDelegate = deposit.amounts - withdrawal.amounts` to finally check the sign of `amount` and act accordingly: withdraw to spot the withdrawal amount, and delegate the remaining.  

This will also have the potential to reduce the gas consumption, as this will lower the number of external calls made to the L1Write contract.

____________________________________
## 🟡 M-02 - `redelegateWithdrawnHYPE` cannot work in the current state

### Summary
The `StakingManager::redelegateWithdrawnHYPE` function is called after `StakingManager::cancelWithdrawal` to redelegate the undelegated tokens of a processed withdrawal. 
The issue is that the function confuse the EVM balance and the HyperCore spot balance, failing to redelegate the tokens.  
### Vulnerability details
First let's see what happens inside `StakingManager::redelegateWithdrawnHYPE`.   
Before executing, the function ensure it has enough HYPE tokens (native token of Hyperliquid) as it can be seen `L948` below.  
But for this balance to exist in the first place, the operator will have to call a function before: `StakingMananger::withdrawFromSpot` [which moves spot tokens back to the EVM](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L993-L997).    
This is mandatory or the `L948` will simply revert, as the withdrawal executed in `processL1Operations` only move the assets from the "staking" balance to the "spot" one.

```solidity
File: src/StakingManager.sol
946:     function redelegateWithdrawnHYPE() external onlyRole(MANAGER_ROLE) whenNotPaused {
947:         require(_cancelledWithdrawalAmount > 0, "No cancelled withdrawals");
948:         require(address(this).balance >= _cancelledWithdrawalAmount, "Insufficient HYPE balance");
949: 
950:         uint256 amount = _cancelledWithdrawalAmount;
951:         _cancelledWithdrawalAmount = 0;
952: 
953:         // Delegate to current validator using the SpotDeposit operation type
954:         _distributeStake(amount, OperationType.SpotDeposit);
955: 
956:         emit WithdrawalRedelegated(amount);
957:     }
```

Then, `L954` it calls `StakingManager::_distributeStake` with `OperationType.SpotDeposit`, leading to this [else block](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L485-L494).
But as we can see, this block only try to ["move the spot balance to the staking balance"](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L490-L491), which will not work as the tokens have been moved back to EVM.  
For this to work, the tokens must first be sent back to spot, as it is done in the above [if block](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L474-L481) in the `OperationType.UserDeposit` case.

We can see that this is a tricky situation with no solution: either the tokens are sent back to EVM for the require statement to pass, but then the staking will not proceed, or the token are not sent back to EVM, and the require statement will revert.
### Impact
Impact is medium as the `redelegateWithdrawnHYPE` cannot work properly, causing the DoS of an important function of the protocol.
Likelihood is medium as canceled withdrawal are not expected to happen that often, but will surely happen once in a while.
### Recommended Mitigation Steps
There is two possibility to solve the issue here.
##### First solution
Rather than reading the "EVM" balance of HYPE token, read the spot balance:
```diff
    function redelegateWithdrawnHYPE() external onlyRole(MANAGER_ROLE) whenNotPaused {
        require(_cancelledWithdrawalAmount > 0, "No cancelled withdrawals");
-       require(address(this).balance >= _cancelledWithdrawalAmount, "Insufficient HYPE balance");
+       (uint64 spotBalance, ,) = L1Read.spotBalance(address(this), HYPE_TOKEN_ID);
+       require(spotBalance >= _cancelledWithdrawalAmount, "Insufficient HYPE spot balance");
 
        uint256 amount = _cancelledWithdrawalAmount;
        _cancelledWithdrawalAmount = 0;

        // Delegate to current validator using the SpotDeposit operation type
        _distributeStake(amount, OperationType.SpotDeposit);

        emit WithdrawalRedelegated(amount);
    }
```

##### Second solution
Send the "EVM" balance back to spot in order to be able to stake it.  
```diff
    function _distributeStake(uint256 amount, OperationType operationType) internal {
	    
	    // ---- some code -------//
	    
        } else if (operationType == OperationType.SpotDeposit) {
            // For spot deposits, first move from spot balance to staking balance
            uint256 truncatedAmount = _convertTo8Decimals(amount, false);
            require(truncatedAmount <= type(uint64).max, "Amount exceeds uint64 max");

+            (bool success, ) = payable(L1_HYPE_CONTRACT).call{value: amount}("");
+            require(success, "Failed to send HYPE to L1");

            // 1. First move from spot balance to staking balance using cDeposit            
            l1Write.sendCDeposit(uint64(truncatedAmount));

	    // ---- some code -------//
```

____________________________________

## 🔵 L-01 - Missing slippage protection on `queueWithdrawal`
_this finding was downgraded from Med to Low_

### Summary
When withdrawing from the `StakingManager`, users provide `kHYPE` shares to be redeemed, which is converted to an equivalent `HYPE` amount using on-chain informations. This means that a user can queue a withdrawal and receive less `HYPE` than he was expecting when sending the transaction (e.g if slashing happens).
### Vulnerability details
For the execution of `queueWithdrawal`, the users provide the amount of `kHYPE` they want to redeem, which is then converted to an amount of HYPE by computing the exchange rate through the [`stakingAccountant::kHYPEToHYPE`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingAccountant.sol#L181-L183) function.  

This exchange rate depends on different values, one of them being the amount of slashed tokens `slashingAmount`.  
If a slashing is reported right before the user `queueWithdrawal` tx is executed, he will suffer a worse exchange rate than expected, without any ways for him to prevent this.  
### Impact
Slippage on withdrawal.  
Impact is high as this represent a direct loss of funds for the user, even though its value will depend on the intensity of the slashing.  
Likelihood is low as slashing events shouldn't happen regularly.  
### Proof of Concept
Please add this PoC to `test/StakingManager.t.sol` :

```solidity
    function testAudit_SlippageOnWithdrawal() public {
        uint256 stakeAmount = 1 ether;

        // setting fees to 0 for readability
        vm.startPrank(manager);
        stakingManager.setUnstakeFeeRate(0);

        // Set up delegation first
        vm.startPrank(manager);
        validatorManager.activateValidator(validator);
        validatorManager.setDelegation(address(stakingManager), validator);
        vm.stopPrank();

        vm.deal(user, stakeAmount);
        vm.prank(user);

        stakingManager.stake{value: stakeAmount}();
        uint256 userShares = kHYPE.balanceOf(user);

        console.log("kHYPE.balanceOf(user): %e\n", userShares);

        uint expectedHYPE = stakingAccountant.kHYPEToHYPE(userShares);

        vm.prank(address(oracleManager));
        validatorManager.reportSlashingEvent(address(validator), 0.1 ether);

        vm.startPrank(user);
        kHYPE.approve(address(stakingManager), userShares);
        stakingManager.queueWithdrawal(userShares);
        vm.stopPrank();

        IStakingManager.WithdrawalRequest memory req = stakingManager.withdrawalRequests(user, 0);
        console.log("expected HYPE to withdraw: %e", expectedHYPE);
        console.log("queued HYPE to withdraw: %e", req.hypeAmount);
    }    
```

### Recommended Mitigation Steps
Update the function to allow users to define a maximum slippage:
```solidity
function queueWithdrawal(uint256 kHYPEAmount, uint256 minHYPEAmount) external nonReentrant whenNotPaused whenWithdrawalNotPaused {
    // ... some code ...
    
    // Convert post-fee kHYPE to HYPE using StakingAccountant
    uint256 hypeAmount = stakingAccountant.kHYPEToHYPE(postFeeKHYPE);
    
    // Add slippage protection
    require(hypeAmount >= minHYPEAmount, "Slippage too high");
    
    // ... rest of function ...
}
```


____________________________________

# ACCEPTED RISK (not rewarded)

## 🟡📩 M-04- `_processL1Withdrawals` should call `L1Read` to ensure correct execution of `L1Write` calls, otherwise they will fail silently

### Summary
The `L1Read` and `L1Write` contracts are system contracts that calls precompiles in order to execute actions or read values on the HyperCore. 
The issue with `L1Write` is that its simply emit events, that are caught by the nodes and executed in non atomically.
This means that any `L1Write` calls can silently fails, while still being accounted as successful by the protocol on the EVM.

### Vulnerability details

Let's see for example the `StakingManager::_processL1Withdrawals` function which [undelegate assets](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L693-L693) to a validator, and [withdraw those assets](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L697-L697) to the "spot" balance.  

If there is a situation where the amount in the "staking" balance are less than `op.amount`, then the call [will be successful](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/lib/L1Write.sol#L30-L30) on the EVM, but will not be executed on the HyperCore (this can be verified on Hyperliquid Testnet which implemented `L1Read` and `L1Write` which I have done through Remix).

One way this can happen is because of the rounding direction that are used when calling `L1Write::sendCDeposit` and `L1Write::sendCWithdrawal`.
We [see in `StakingManager::_distributeStake`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L487-L491) that before calling `sendCDeposit`, the amount is converted to 8 decimals and rounded down.
But in `StakingManager::_withdrawFromValidator`, the amount to withdraw is [converted to 8 decimals **then rounded up**](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L542-L542).
This happens by calling the [`_convertTo8Decimals`](https://github.com/code-423n4/2025-04-kinetiq/blob/main/src/StakingManager.sol#L415-L427) function.
This make it possible to have a difference off by 1 for a same value, where the deposited amount will be 1 wei lower than the withdrawn amount.
E.g, if `amount = 1e10 + 1` (which is possible in the `OperationType.SpotDeposit` case, created through `StakingManager::redelegateWithdrawnHYPE` when a withdrawal has been canceled), the deposited amount will be equal to converted and rounded down amount will be equal to `1`.
If the same amount is then withdrawn again (e.g if the user which has his first withdrawal canceled), the result of `_convertTo8Decimals` will be `2`.

This one case is easy to find, but there might be other situations where this might come to play, thus it is important to add safeguards to ensure correct execution of `L1Write` calls.
### Impact
Non execution of `L1Write` calls on the HyperCore, messing up with staking operations of the `StakingManager`.
### Recommended Mitigation Steps
Add sanity checks using `L1Read` contract before L1 operations.

