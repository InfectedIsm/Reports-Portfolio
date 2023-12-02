# DittoETH - Findings Report (September 2023)

# Table of contents

- ## High Risk Findings
    - ### [H-01. Owner of a bad ShortRecord can front-run flagShort calls AND liquidateSecondary and prevent liquidation](#H-01)
- ## Medium Risk Findings
    - ### [M-01. unstaking rETH could not work if delay is set again](#M-01)
- ## Low Risk Findings
    - ### [L-01. BridgeRouterFacet::withdraw() and unstake() can revert when amount * TVL > uint88 because of PRBMathHelper::mulU88](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 1
   - Low: 1


# High Risk Findings (Selected For Report)

## <a id='H-01'></a>H-01. Owner of a bad ShortRecord can front-run flagShort calls AND liquidateSecondary and prevent liquidation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/MarginCallPrimaryFacet.sol#L47

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/AppStorage.sol#L101

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ERC721Facet.sol#L162C17-L162C17

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L132

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L224C10-L224C10

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L47C9-L47C9

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallSecondaryFacet.sol#L64C25-L64C25

## Summary

A shorter can keep a unhealthy short position open by minting an NFT of it and front-running attempts to liquidate it with a transfer of this NFT (which transfers the short position to the new owner)

## Vulnerability Details

A Short Record (SR) is a struct representing a short position that has been opened by a user.
It holds different informations, such as how much collateral is backing the short, and how much debt it owe (this ratio is called Collateral Ratio or CR)
At any time, any user can flag someone's else SR as "dangerous", if its debt grows too much compared to its collateral.
This operation is accessible through `MarginCallPrimaryFacet::flagShort`, [which check through the `onlyValidShortRecord` modifier](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L47C9-L47C9)  that the [SR isn't `Cancelled`]((https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/MarginCallPrimaryFacet.sol#L47))
If the SR is valid, then its debt/collateral ratio is verified, and if its below a specific threshold, flagged.
But that also means that if a SR is considered invalid, it cannot be flagged.
And it seems there is a way for the owner of a SR to cancel its SR while still holding the position.

The owner of a SR can mint an NFT to represent it and make it transferable.
This is done in 5 steps:
1) `TransferFrom` verify usual stuff regarding the NFT (ownership, allowance, valid receiver...)
2) `LibShortRecord::transferShortRecord` [is called](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ERC721Facet.sol#L162C17-L162C17)
3) [`transferShortRecord` verify](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/MarginCallPrimaryFacet.sol#L47) that SR [is not `flagged` nor `Cancelled`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/AppStorage.sol#L101)
4) SR is [deleted](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L132) ([setting its status](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L224C10-L224C10) to `Cancelled`)
5) a new SR is created with same parameters, but owned by the receiver.

Now, let's see what would happen if Alice has a SR_1 with a bad CR, and Bob tries to flag it.

1. Bob calls `flagShort`on SR_1, the tx is sent to the mempool

Alice is watching the mempool, and don't want her SR to be flagged:

2. She front-run Bob's tx with a transfer of her SR_1 to another of the addresses she controls

Now Bob's tx will be executed after Alice's tx:

3) The SR_1 is "deleted" and its status set to `Cancelled`
4) Bob's tx is executed, and `flagShort` reverts because of the [`onlyValidShortRecord`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L47C9-L47C9)
5) Alice can do this trick again to keep here undercol SR until it can become dangerous

But this is not over:

6) Even when her CR drops dangerously (CR<1.5), `liquidateSecondary` is also DoS'd as it has [the same check](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallSecondaryFacet.sol#L64C25-L64C25) for `SR.Cancelled` 

## Impact

Because of this, a shorter could maintain the dangerous position (or multiple dangerous positions), while putting the protocol at risk.

## Proof of Concept

Add these tests to `ERC721Facet.t.sol` :

#### Front-running flag
```solidity
	function test_audit_frontrunFlagShort() public {
		address alice = makeAddr("Alice"); //Alice will front-run Bob's attempt to flag her short
		address aliceSecondAddr = makeAddr("AliceSecondAddr");
		address bob = makeAddr("Bob"); //Bob will try to flag Alice's short 
		address randomUser = makeAddr("randomUser"); //regular user who created a bid order
		
		//A random user create a bid, Alice create a short, which will match with the user's bid
		fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, randomUser);
		fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, alice);
		//Alice then mint the NFT associated to the SR so that it can be transfered
		vm.prank(alice);
		diamond.mintNFT(asset, Constants.SHORT_STARTING_ID);

		//ETH price drops from 4000 to 2666, making Alice's short flaggable because its < LibAsset.primaryLiquidationCR(asset)
		setETH(2666 ether);
		
		// Alice saw Bob attempt to flag her short, so she front-run him and transfer the SR
		vm.prank(alice);
		diamond.transferFrom(alice, aliceSecondAddr, 1);
		
		//Bob's attempt revert because the transfer of the short by Alice change the short status to SR.Cancelled
		vm.prank(bob);
		vm.expectRevert(Errors.InvalidShortId.selector);
		diamond.flagShort(asset, alice, Constants.SHORT_STARTING_ID, Constants.HEAD);
	}	
```

#### Front-running liquidateSecondary
```solidity
    function test_audit_frontrunPreventFlagAndSecondaryLiquidation() public {
		address alice = makeAddr("Alice"); //Alice will front-run Bob's attempt to flag her short
		address aliceSecondAddr = makeAddr("AliceSecondAddr");
		address aliceThirdAddr = makeAddr("AliceThirdAddr");
		address bob = makeAddr("Bob"); //Bob will try to flag Alice's short 
		address randomUser = makeAddr("randomUser"); //regular user who created a bid order
		
		//A random user create a bid, Alice create a short, which will match with the user's bid
        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, randomUser);
		fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, alice);
		//Alice then mint the NFT associated to the SR so that it can be transfered
		vm.prank(alice);
        diamond.mintNFT(asset, Constants.SHORT_STARTING_ID);

        //set cRatio below 1.1
        setETH(700 ether);
		
		//Alice is still blocking all attempts to flag her short by transfering it to her secondary address by front-running Bob
        vm.prank(alice);
        diamond.transferFrom(alice, aliceSecondAddr, 1);
		vm.prank(bob);
		vm.expectRevert(Errors.InvalidShortId.selector);
		diamond.flagShort(asset, alice, Constants.SHORT_STARTING_ID, Constants.HEAD);

		//Alice  front-run (again...) Bob and transfers the NFT to a third address she owns
		vm.prank(aliceSecondAddr);
        diamond.transferFrom(aliceSecondAddr, aliceThirdAddr, 1);

		//Bob's try again on the new address, but its attempt revert because the transfer of the short by Alice change the short status to SR.Cancelled
		STypes.ShortRecord memory shortRecord = getShortRecord(aliceSecondAddr, Constants.SHORT_STARTING_ID);
		depositUsd(bob, shortRecord.ercDebt);
        vm.expectRevert(Errors.MarginCallSecondaryNoValidShorts.selector);
		liquidateErcEscrowed(aliceSecondAddr, Constants.SHORT_STARTING_ID, DEFAULT_AMOUNT, bob);
    }
```

## Tools Used

Manual review

## Recommended Mitigation Steps

SR with a bad CR shouldn't be transferable, the user should first make its position healthy before being allowed to transfer it.
I suggest to add a check in `LibShortRecord::transferShortRecord` to ensure `CR > primaryLiquidationCR

```shell
diff --git a/contracts/libraries/LibShortRecord.sol b/contracts/libraries/LibShortRecord.sol
index 7c5ecc3..8fad274 100644
--- a/contracts/libraries/LibShortRecord.sol
+++ b/contracts/libraries/LibShortRecord.sol
@@ -15,6 +15,7 @@ import {LibOracle} from "contracts/libraries/LibOracle.sol";
 // import {console} from "contracts/libraries/console.sol";

 library LibShortRecord {
+    using LibShortRecord for STypes.ShortRecord;
     using U256 for uint256;
     using U88 for uint88;
     using U80 for uint80;

@@ -124,10 +125,16 @@ library LibShortRecord {
         uint40 tokenId,
         STypes.NFT memory nft
     ) internal {

+        AppStorage storage s = appStorage();
         STypes.ShortRecord storage short = s.shortRecords[asset][from][nft.shortRecordId];
         if (short.status == SR.Cancelled) revert Errors.OriginalShortRecordCancelled();
         if (short.flaggerId != 0) revert Errors.CannotTransferFlaggedShort();
+               if (
+                       short.getCollateralRatioSpotPrice(LibOracle.getSavedOrSpotOraclePrice(asset))
+                               < LibAsset.primaryLiquidationCR(asset)
+        ) {
+            revert Errors.InsufficientCollateral();
+        }

         deleteShortRecord(asset, from, nft.shortRecordId);

```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. unstaking rETH could not work if delay is set again            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/bridges/BridgeReth.sol#L102

https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L157-L172

## Summary
rETH tokens has implemented an "unstake delay" to prevent sandwich attack. This delay has been set to 0 and replaced by fees, but if for any reason it is set back to a non zero value, users will not be anymore able to unstake from the bridge.

## Vulnerability Details

The protocol allow users who deposited rETH into the bridge to either withdraw it or unstake it:
```solidity
    // Exchange system rETH to fulfill zETH obligation to user
    function withdraw(address to, uint256 amount)
        external
        onlyDiamond
        returns (uint256)
    {
        IRocketTokenRETH rocketETHToken = _getRethContract();
        // Calculate zETH equivalent value in rETH
        uint256 rethValue = rocketETHToken.getRethValue(amount);
        // Transfer rETH from this bridge contract
        // @dev RETH uses OZ ERC-20, don't need to check success bool
        rocketETHToken.transfer(to, rethValue);
        return rethValue;
    }


    function unstake(address to, uint256 amount) external onlyDiamond {
        IRocketTokenRETH rocketETHToken = _getRethContract();
        uint256 rethValue = rocketETHToken.getRethValue(amount);
        uint256 originalBalance = address(this).balance;
        rocketETHToken.burn(rethValue);
        uint256 netBalance = address(this).balance - originalBalance;
        if (netBalance == 0) revert NetBalanceZero();
        (bool sent,) = to.call{value: netBalance}("");
        assert(sent);
    }

```


The thing is, RocketPool rETH tokens have a deposit delay parameter that prevents any user who has recently deposited to transfer or burn tokens:

```solidity
// https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L156-L172

// This is called by the base ERC20 contract before all transfer, mint, and burns
function _beforeTokenTransfer(address from, address, uint256) internal override {
	// Don't run check if this is a mint transaction
	if (from != address(0)) {
		// Check which block the user's last deposit was
		bytes32 key = keccak256(abi.encodePacked("user.deposit.block", from));
		uint256 lastDepositBlock = getUint(key);
		if (lastDepositBlock > 0) {
			// Ensure enough blocks have passed
			uint256 depositDelay = getUint(keccak256(abi.encodePacked(keccak256("dao.protocol.setting.network"), "network.reth.deposit.delay")));
			uint256 blocksPassed = block.number.sub(lastDepositBlock);
			require(blocksPassed > depositDelay, "Not enough time has passed since deposit");
			// Clear the state as it's no longer necessary to check this until another deposit is made
			deleteUint(key);
		}
	}
}
```

In the past this delay was set to 5760 blocks mined (aprox. 21h, considering one block per 12s), but it has been set to 0, and replaced by fees instead (the delay was set [to mitigate sandwich attack](https://dao.rocketpool.net/t/remove-reth-lock-after-mint/339))
While it's not currently possible due to RocketPool's configuration, any future changes made to this delay by the admins could potentially lead to a denial-of-service attack on the `unstake()` mechanism. 

After depositing, a user must wait a delay before being able to withdraw its stake
But here, its the bridge contract who deposit, which mean each time a user deposit through the bridge,
the delay is reset.

As there is a possibility for user to withdraw their rETH directly without unstaking, but still can possibly make it impossible to use the function, I classified it as medium severity.

in the `unstake` function, `rocketTokenRETH.burn` is called
Which itself `_beforeTokenTransfer`, and this one is subject to a delay depending on last deposit

## Impact
Users will not be able to unstake from the bridge.

## Tools Used
Manual review 

## Recommended Mitigation Steps
Check that the timelock period will not make revert the withdrawal and, if so, use a DEX to convert the rETH to ETH.

# Low Risk Findings

## <a id='L-01'></a>L-01. BridgeRouterFacet::withdraw() and unstake() can revert when amount * TVL > uint88 because of PRBMathHelper::mulU88            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L110

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L136

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L184

https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/PRBMathHelper.sol#L102

## Summary

When the protocol will be well established, there might be user wanting to withdraw substantial amount of ETH, but the call will revert because of an overflow that shouldn't happen.

## Vulnerability Details

In [`withdraw()`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L110) or [`unstakeEth()`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L136), the amount that the user want to get is converted from the LSD to ETH by calling the function [`_ethConversion(...)`](https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BridgeRouterFacet.sol#L184) :

```solidity
    function _ethConversion(uint256 vault, uint88 amount) private view returns (uint88) {
        uint256 zethTotalNew = vault.getZethTotal();
        uint88 zethTotal = s.vault[vault].zethTotal;


        if (zethTotalNew >= zethTotal) {
            // when yield is positive 1 zeth = 1 eth
            return amount;
        } else {
            // negative yield means 1 zeth < 1 eth
            return amount.mulU88(zethTotalNew).divU88(zethTotal); //@audit revert if amount*zethTotalNew > u88, replace mulU88 by mul
        }
    }
```

But as we can see in the else case, `amount` is multiplied by `zethTotalNew` using the `PRBMathHelper::mulU88` function.
This function reverts when the result of the calculation is greater than U88.
But there is realistic situations (*amount: 333 ETH*, *zethTotalNew: 1M ETH*, [see total LSD staked here](https://defillama.com/lsd): 11.3M ETH staked) where this can happen.
The thing is that what matters in this calculation, is that the final result fits in a U88.
That means, only the division should check the overflow, but its not the case.

## Impact

User request to withdraw or unstake will revert.

## Proof of Concept

Add this test to `Yield.t.sol`:
```solidity
function test_audit_withdrawInvalidAmount() public {
		uint88 revertWithdrawal = 333 ether;
		uint88 zethTotal = 1_000_000 ether;
		uint88 zethTotalNew = 1_000_001 ether;
	
        deal(_reth, sender, revertWithdrawal+100);
        deal(_reth, extra, zethTotal);

		vm.prank(extra);
		diamond.deposit(_bridgeReth, zethTotal);

		vm.prank(sender);
		diamond.deposit(_bridgeReth, revertWithdrawal+100);

		deal(_reth, _bridgeReth, zethTotalNew); // Mimics loss of value of 0.1 ETH

		vm.prank(sender);
		diamond.withdraw(_bridgeReth, revertWithdrawal);		
	}
```
## Tools Used

Manual review

## Recommended Mitigation Steps

replace the first mulU88 by a mul:

```shell
diff --git a/contracts/facets/BridgeRouterFacet.sol b/contracts/facets/BridgeRouterFacet.sol
index c9ff4e5..3680ae6 100644
--- a/contracts/facets/BridgeRouterFacet.sol
+++ b/contracts/facets/BridgeRouterFacet.sol
 
@@ -176,12 +176,12 @@ contract BridgeRouterFacet is Modifiers {
	    function _ethConversion(uint256 vault, uint88 amount) private view returns (uint88) {
         uint256 zethTotalNew = vault.getZethTotal();
         uint88 zethTotal = s.vault[vault].zethTotal;
 
        if (zethTotalNew >= zethTotal) { 
             // when yield is positive 1 zeth = 1 eth
             return amount;
         } else {
             // negative yield means 1 zeth < 1 eth
-            return amount.mulU88(zethTotalNew).divU88(zethTotal);
+            return amount.mul(zethTotalNew).divU88(zethTotal); //@audit-issue revert if amount*zethTotalNew > u88, replace mulU88 by mul
         }
     }
```


