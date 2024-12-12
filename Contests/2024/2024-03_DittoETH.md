

# 🟡M01 - If a redemption has N disputable shorts, it is possible to dispute N-1 times the redemption to maximize the penalty

Ditto is a decentralized CDP protocol allowing users to get stablecoins (called dUSD here) against LST (Liquid Staking Tokens like stETH or rETH) collateral. 
Depositing LST grant dETH, a shares representing the value deposited in ETH.
Users can then open different kind of positions: bids, asks or shorts, each one having its own orderbook.

Ditto implemented a mechanism called Redemption, idea borrowed from [Liquity](https://docs.liquity.org/faq/lusd-redemptions)
Redemption allow anyone to redeem dUSD for a value of exactly 1 USD by calling out shorts with very bad CR (collateral ratio) and closing them.
This helps the protocol to keep a healthy global market CR, shorters to not be liquidated, and dUSD holders to redeem dUSD with no loss in case of dUSD trading below 1 USD.

Users can propose multiple shorts to be redeemed at once if they meet certain conditions ([see Constraints in doc](https://dittoeth.com/technical/redemptions#proposing-a-redemption))
The shorts must be sorted from lowest CR to highest CR, and all CRs must be below a CR of 2.
If a proposal do not properly follow these rules, anyone can dispute the proposal by showing a proof-short: all proposal shorts with a CR higher than the proof become invalid.
For each invalid short, a penalty is applied to the redeemer (and awarded to the disputer)
The penalty calculation is based on the CR difference between the disputeCR (proof) and the the currentProposal.CR (lowest invalid CR) :

1)
$$
\text{penaltyPct} = \min \left( \max \left( \text{callerFeePct}, \frac{\text{currentProposal.CR} - \text{disputeCR}}{\text{currentProposal.CR}} \right), 0.33 \, \text{ether} \right);
$$	

2.
$$
\text{penaltyAmt} = \text{incorrectErcDebt} \cdot \text{penaltyPct};
$$


The issue is located in this part of the mechanism and is pretty easy to understand.
Let's imagine 4 shorts in a list of shorts:  ... < CR1 < CR2 < CR3 < CR4 < ...
CR1 = 1.1, CR2 = 1.2, CR3 = 1.3, CR4 = 1.4, and all have an ercDebt of 1 ETH
Redeemer propose [CR2, CR3, CR4] to redeem.
Disputer sees that there is in fact CR1 that is lower than all the proposed shorts.
Disputer could dispute the proposal by giving CR1 as a proof, and CR2 as the invalid CR, which would also invalid all higher CRs in the proposal.
This would give (let's take callerFeePct value from tests, 0.005 or 0.5%)
1. `penaltyPct =` ` min( max(0.005, (1.2 - 1.1)/1.2 ), 0.33) = min( max(0.005, 0.091), 0.33) =` ` 0.091 `
2. `penaltyAmt =` `(1 + 1 + 1)ETH * 0.091 = 3 * 0.091 =` `0.273 ETH`

The vulnerability lies in the fact that the disputer can dispute this proposal 3 times:
1st: dispute CR4 with CR1
2nd: dispute CR3 with CR1
3rd: dispute CR2 with CR1

By doing this, the disputer will get penalty applied 3 times, and in some case, the total penalty using this trick will be higher than the penalty when disputing the whole proposal at once, as shown in the coded proof of concept.

For CR1:
1. `penaltyPct =` ` min( max(0.005, (1.4 - 1.1)/1.4 ), 0.33) =` ` 0.214 `
2. `penaltyAmt =` ` 1 ETH * 0.214 =` `0.214 ETH`

For CR2:
1. `penaltyPct =` ` min( max(0.005, (1.3 - 1.1)/1.3 ), 0.33) =` ` 0.142 `
2. `penaltyAmt =` ` 1 ETH * 0.214 =` `0.142 ETH`

For CR3:
1. `penaltyPct =` ` min( max(0.005, (1.2 - 1.1)/1.2 ), 0.33) =` ` 0.091 `
2. `penaltyAmt =` ` 1 ETH * 0.091 =` `0.091 ETH`

- `sum of penaltyAmt =` ` 0.447 ETH`

We can see here how more beneficial it to adopt the second strategy.

## Impact
Higher penalty than expected for redeemer, higher reward for disputer.

## Proof of Concept

Copy [this gist](https://gist.github.com/InfectedIsm/036947553bdfc984a19b929eea4aade5) to `test/Redemption.t.sol` and run the tests by executing `forge test -vvv --mt AuditDisputeRedemption`
You'll get that output, showing that disputing redemptions one by one give a higher penalty: 

```
Running 2 tests for test/Redemption.t.sol:RedemptionTest
[PASS] test_AuditDisputeRedemptionAllAtOnce() (gas: 2213036)
Logs:
  redeemer reward: 7.69230769230769231e21

[PASS] test_AuditDisputeRedemptionOneByOne() (gas: 2250699)
Logs:
  redeemer reward: 3.846153846153846155e21
  redeemer reward: 4.166666666666666665e21
```


## Tools Used
Manual review

## Recommended Mitigation Steps
If a user gives N as the incorrectIndex in the disputed proposal, knowing that the proposal is sorted from lowest to highest CR, ensure that `proposal[N-1].CR <= disputeCR` (revert in that case)


-----------------------------------------

# 🟡 M02 - Wrong decimals in `LibOracle::getOraclePrice` for `oracle != baseOracle`

C.BASE_ORACLE_DECIMALS stores the difference between the base oracle price decimals and ditto internal decimals.
While this is use correctly in the other places of the function, the last case (`baseOracle.latestRoundData()` fails and `oracle != baseOracle`) uses the wrong constant to normalize the oracle price.

```solidity
File: contracts\libraries\LibOracle.sol

19:     function getOraclePrice(address asset) internal view returns (uint256) {
20:         AppStorage storage s = appStorage();
21:         AggregatorV3Interface baseOracle = AggregatorV3Interface(s.baseOracle);
22:         uint256 protocolPrice = getPrice(asset);
23: 
24:         AggregatorV3Interface oracle = AggregatorV3Interface(s.asset[asset].oracle);
25:         if (address(oracle) == address(0)) revert Errors.InvalidAsset();
26: 
27:         try baseOracle.latestRoundData() returns (uint80 baseRoundID, int256 basePrice, uint256, uint256 baseTimeStamp, uint80) {
28:             if (oracle == baseOracle) {
29:                 // @dev multiply base oracle by 10**10 to give it 18 decimals of precision
30:                 uint256 basePriceInEth = basePrice > 0 ? uint256(basePrice * C.BASE_ORACLE_DECIMALS).inv() : 0;
31:                 basePriceInEth = baseOracleCircuitBreaker(protocolPrice, baseRoundID, basePrice, baseTimeStamp, basePriceInEth);
32:                 return basePriceInEth;
... :		// ...
... :		// ...
47:         } catch {
48:             if (oracle == baseOracle) {
49:                 return twapCircuitBreaker();
50:             } else {
51:                 // prettier-ignore
52:                 (
53:                     uint80 roundID,
54:                     int256 price,
55:                     /*uint256 startedAt*/
56:                     ,
57:                     uint256 timeStamp,
58:                     /*uint80 answeredInRound*/
59:                 ) = oracle.latestRoundData();	//<@ oracle is called, not baseOracle
60:                 if (roundID == 0 || price == 0 || timeStamp > block.timestamp) revert Errors.InvalidPrice();
61: 
62:                 uint256 twapInv = twapCircuitBreaker();
63:❌		        uint256 priceInEth = uint256(price * C.BASE_ORACLE_DECIMALS).mul(twapInv);//this oracle could have different decimals
64:                 return priceInEth;
65:             }
66:         }
67:     }
```
The `mul` function from [PRBMathHelper](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/PRBMathHelper.sol#L12-L14) calls the [`mulDiv18` function](https://github.com/PaulRBerg/prb-math/blob/a111d11988756e091c7ccfd1f5af490d4ddf8783/src/Common.sol#L495) 
Which calculate: `priceInEth = price * C.BASE_ORACLE_DECIMALS * twapInv / 1e18`
`twapInv` return a [18 decimal number](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibOracle.sol#L141) which is correctly cancelled by the `/1e18` of `mul`
This means at the end, for `priceInEth` to have the expected 18 decimals, **price must be multiplied by the correct exponent**, and `C.BASE_ORACLE_DECIMALS` might not be correct in some cases.

There is a [similar issue L43](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibOracle.sol#L43) of the same contract, where it is assumed that price and basePrice will have same decimals, while this could be different.

```solidity
File: contracts\libraries\LibOracle.sol
27:         try baseOracle.latestRoundData() returns (uint80 baseRoundID, int256 basePrice, uint256, uint256 baseTimeStamp, uint80) {
28:             if (oracle == baseOracle) {
29:                 // @dev multiply base oracle by 10**10 to give it 18 decimals of precision
30:                 uint256 basePriceInEth = basePrice > 0 ? uint256(basePrice * C.BASE_ORACLE_DECIMALS).inv() : 0;
31:                 basePriceInEth = baseOracleCircuitBreaker(protocolPrice, baseRoundID, basePrice, baseTimeStamp, basePriceInEth);
32:                 return basePriceInEth;
33:             } else {
34:                 // prettier-ignore
35:                 (
36:                     uint80 roundID,
37:                     int256 price,
38:                     /*uint256 startedAt*/
39:                     ,
40:                     uint256 timeStamp,
41:                     /*uint80 answeredInRound*/
42:                 ) = oracle.latestRoundData();
43:❌	            uint256 priceInEth = uint256(price).div(uint256(basePrice)); //same issue can happen with decimals here
44:                 oracleCircuitBreaker(roundID, baseRoundID, price, basePrice, timeStamp, baseTimeStamp);
45:                 return priceInEth;
46:             }
```

The `div` function from [PRBMathHelper](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/PRBMathHelper.sol#L16-L18) calls the [`mulDiv` function](https://github.com/PaulRBerg/prb-math/blob/a111d11988756e091c7ccfd1f5af490d4ddf8783/src/Common.sol#L387) 
Which at the end does : `priceInEth = price * 1e18 / basePrice`
If `price` and `basePrice` have the same decimals, then it works and `priceInEth` has 18 decimals.
But if they differ in decimals, then princeInEth will have an unexpected number of decimals.

## Impact
Few decimals of error can have a huge impact on the resulting value calculation.
Calculated price will be wrong and will impact many functions of the protocol (deposit/withdraw, liquidation, redeem, ...)
This could have a huge impact on users and the protocol itself.

## Tools Used
Manual review

## Recommended Mitigation Steps
Either save the decimals difference in a mapping for each oracle, or read it from the oracle itself using the [`AggregatorV3Interface::decimals()` function](https://docs.chain.link/data-feeds/api-reference#functions-in-aggregatorv3interface)




-----------------------------------------

# 🟠M03 - If there is less than 255 opened shorts and all are proposed in one proposal, its not possible to dispute it

https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L62
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L244

## Summary
If there is less than 255 opened shorts, a malicious user can create a proposal containing all of them, making his proposal undisputable.
This will force all shorters to exit their positions.

## Vulnerability details

<details close>
  <summary><b>click for context</b></summary>
Ditto is a decentralized CDP protocol allowing users to get stablecoins (called dUSD here) against LST (Liquid Staking Tokens like stETH or rETH) collateral. 
Depositing LST grant dETH, a shares representing the value deposited in ETH.
Users can then open different kind of positions: bids, asks or shorts, each one having its own orderbook.

Ditto implemented a mechanism called Redemption, idea borrowed from [Liquity](https://docs.liquity.org/faq/lusd-redemptions)
Redemption allow anyone to redeem dUSD for a value of exactly 1 USD by calling out shorts with very bad CR (collateral ratio) and closing them.
This helps the protocol to keep a healthy total CR, shorters to not be liquidated, and dUSD holders to redeem dUSD with no loss in case of dUSD trading below 1 USD.

Users can propose multiple shorts to be redeemed at once if they meet certain conditions ([see Constraints in doc](https://dittoeth.com/technical/redemptions#proposing-a-redemption))
The shorts must be sorted from lowest CR to highest CR, and all CRs must be below a CR of 2.
If a proposal do not properly follow these rules, anyone can dispute the proposal by showing a proof-short: all proposal shorts with a CR higher than the proof become invalid.
</details>

A proposal can have as much as [255 Short Records (SR)](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L62).
To dispute a proposal, the disputer must give only one short that has a lower CR than the ones in the proposal, and obviously [not part of the proposal](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L244).
The other important condition here is that the dispute SR can't be newer than the proposal. 
But what would happen at the very beginning of the protocol, when there are less than 255 shorts?
Malicious users could create a proposal with all valid and existing shorts. And because it is impossible to use a short from the proposal to dispute, the proposal cannot be disputed and will pass.


## Impact
This will disrupt the redemption and short mechanism of Ditto.

## Proof of Concept

short limit in `proposeRedemption`:
```solidity
File: contracts\facets\RedemptionFacet.sol

56:     function proposeRedemption(
57:         address asset,
58:         MTypes.ProposalInput[] calldata proposalInput,
59:         uint88 redemptionAmount,
60:         uint88 maxRedemptionFee
61:     ) external isNotFrozen(asset) nonReentrant {
62:⚠       if (proposalInput.length > type(uint8).max) revert Errors.TooManyProposals();  //<@ limit to 255 shorts
```

Disputes will revert if all existing shorts are in a proposal:
```solidity
File: contracts\facets\RedemptionFacet.sol
224:  function disputeRedemption(address asset, address redeemer, uint8 incorrectIndex, address disputeShorter, uint8 disputeShortId)
225:         external
226:         isNotFrozen(asset)
227:         nonReentrant
228:     {
229:         if (redeemer == msg.sender) revert Errors.CannotDisputeYourself();
230:         MTypes.DisputeRedemption memory d;
231:         d.asset = asset;
232:         d.redeemer = redeemer;
233: 
234:         STypes.AssetUser storage redeemerAssetUser = s.assetUser[d.asset][d.redeemer];
235:         if (redeemerAssetUser.SSTORE2Pointer == address(0)) revert Errors.InvalidRedemption();
236: 
237:         if (LibOrders.getOffsetTime() >= redeemerAssetUser.timeToDispute) revert Errors.TimeToDisputeHasElapsed();
238: 
239:         MTypes.ProposalData[] memory decodedProposalData =
240:             LibBytes.readProposalData(redeemerAssetUser.SSTORE2Pointer, redeemerAssetUser.slateLength);
241: 
242:         for (uint256 i = 0; i < decodedProposalData.length; i++) {
243:             if (decodedProposalData[i].shorter == disputeShorter && decodedProposalData[i].shortId == disputeShortId) {
244:❌				 revert Errors.CannotDisputeWithRedeemerProposal(); //<@ if all existing shorts are in proposal, any dispute will revert
245:             }
246:         }
```


## Tools Used
Manual review

## Recommended Mitigation Steps
In my opinion, there is no need to allow as much as 255 shorts in a proposal. 
I would suggest to decrease this limit to a lower value to make this exploit less likely to happen.
Another way to do this would be to keep a count of active shorts, and prevent users to redeem more than X% of active shorts.

-----------------------------------------

# 🟠M04 - Using cached price to create a proposal reduce the efficacity of redemptions for asset peg

One of the conditions for updating oracle prices in Ditto is [when an action related to shorts is executed](https://dittoeth.com/technical/oracles#conditions-for-updating-oracle).
This is important because, in the event of high volatility, shorts must be closed out before bad debt occurs in the protocol.
While [liquidations does update the oracle](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/PrimaryLiquidationFacet.sol#L68-L68) before processing the short, this is not the case for [redemptions](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L75)

Redemptions, as liquidations, play a central role in Ditto. By allowing users to redeem shorts with with poor collateralization for a 1:1 exchange rate of the asset, Ditto is able to maintain a stable peg for its pegged asset.
For this reason, it is important to reduce the pricing delay for the redeems, as much as for the liquidations.

```solidity
File: contracts\facets\RedemptionFacet.sol
56:     function proposeRedemption(
57:         address asset,
58:         MTypes.ProposalInput[] calldata proposalInput,
59:         uint88 redemptionAmount,
60:         uint88 maxRedemptionFee
61:     ) external isNotFrozen(asset) nonReentrant {
...:
...:		//...
74: 
75:⚠		p.oraclePrice = LibOracle.getPrice(p.asset); //<@ getting cached price
76: 
77:         bytes memory slate;
78:         for (uint8 i = 0; i < proposalInput.length; i++) {
79:             p.shorter = proposalInput[i].shorter;
80:             p.shortId = proposalInput[i].shortId;
81:             p.shortOrderId = proposalInput[i].shortOrderId;
82:             // @dev Setting this above _onlyValidShortRecord to allow skipping
83:             STypes.ShortRecord storage currentSR = s.shortRecords[p.asset][p.shorter][p.shortId];
84: 
85:             /// Evaluate proposed shortRecord
86: 
87:             if (!validRedemptionSR(currentSR, msg.sender, p.shorter, minShortErc)) continue;
88: 
89:             currentSR.updateErcDebt(p.asset);
90:⚠			p.currentCR = currentSR.getCollateralRatio(p.oraclePrice); //<@ Collateral ratio calculated using cached price
91: 
92:             // @dev Skip if proposal is not sorted correctly or if above redemption threshold
93:❌			if (p.previousCR > p.currentCR || p.currentCR >= C.MAX_REDEMPTION_CR) continue; //<@ 
```

## Impact

If the cached price is not reflective of the current market price, the protocol may either overvalue or undervalue the collateral backing shorts.
The usage of cached prices in the proposeRedemption function, as opposed to real-time or recently updated prices will affect the effectiveness of the redemption process in maintaining the asset's peg in periods of high volatility. 


## Tools Used
Manual review

## Recommended Mitigation Steps
Do not use cached price for redemptions, but rather an updated price through `LibOracle::getSavedOrSpotOraclePrice` for example.

-----------------------------------------

# 

## Impact
Detailed description of the impact of this finding.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

## Tools Used

## Recommended Mitigation Steps



-----------------------------------------

# Accepted risk

# 🔴 H01 - transfering a SR will close it, making it invalid for redemption

## _this finding was aknowledged as protocol will use flashbots

Ditto is a decentralized CDP protocol allowing users to get stablecoins (called dUSD here) against LST (Liquid Staking Tokens like stETH or rETH) collateral. 
Depositing LST grant dETH, a shares representing the value deposited in ETH.
Users can then open different kind of positions: bids, asks or shorts, each one having its own orderbook.

When a short is opened, an empty Short Record (SR) is created, linked to the short order and to the shorter address.
From a short record, an NFT can be minted, making it transferable.

[Transfering a SR consist in fact](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L139-L144) in deleting the SR, transfering the assets associated to this short to the new owner, and create a new SR. 
Deleting a SR [sets its status to `SR.Closed`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/LibSRUtil.sol#L139-L144)

And this is the issue. The ability for a SR record owner to transfer its SR, and this changing its status, open up the possbility to escape redemptions and liquidations by front-running the calls.

This is because the [`RedemptionFacet::proposeRedemption` function before processing a short](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts\facets\RedemptionFacet.sol#L87), verify that the SR [has not been closed](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts\facets\RedemptionFacet.sol#L38)

Same idea for liquidations, where [`PrimaryLiquidationFacet::liquidate`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/PrimaryLiquidationFacet.sol#L51-L51) through its [`onlyValidaShortRecord` modifier](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/libraries/AppStorage.sol#L86-L98) will revert in case if frontrun by a transfered SR.
Finaly, we see the same behavior here on [`SecondaryLiquidation::liquidateSecondary`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/SecondaryLiquidationFacet.sol#L63-L63)

## Impact
It is possible for a shorter to interfer with vital mechanism of Ditto ETH (redemptions/liquidation)

## Proof of Concept

Escaping redeeming:

```solidity
	function test_audit_EscapeRedeem() public { 
		address alice = sender; //Alice will front-run Bob's attempt to flag her short
		vm.label(alice, "Alice");
		address bob = receiver; //Bob will try to flag Alice's short 
		vm.label(bob, "Bob");
		address aliceSecondAddr = makeAddr("AliceSecondAddr");
		
		//A random user create a bid, Alice create a short, which will match with the user's bid
		fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, bob);
        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, alice);
		//Alice then mint the NFT associated to the SR so that it can be transfered
		vm.prank(alice);
		diamond.mintNFT(asset, C.SHORT_STARTING_ID, 0);

		//ETH price drops from 4000 to 1500, making her position redeemable
		setETH(1500 ether);

		MTypes.ProposalInput[] memory proposalInputs = new MTypes.ProposalInput[](1);
        proposalInputs[0] = MTypes.ProposalInput({shorter: sender, shortId: C.SHORT_STARTING_ID, shortOrderId: 0});

		// Alice sees Bob attempt to liquidate her short, so she front-run him and transfer the SR
		vm.prank(alice);
		diamond.transferFrom(alice, aliceSecondAddr, 1);

        vm.prank(bob);
		vm.expectRevert(Errors.RedemptionUnderMinShortErc.selector);
        diamond.proposeRedemption(asset, proposalInputs, DEFAULT_AMOUNT, MAX_REDEMPTION_FEE);
	}	
```

Escaping liquidation:

```solidity
	function test_audit_EscapeLiquidation() public { 
		address alice = makeAddr("Alice"); //Alice will front-run Bob's attempt to flag her short
		address aliceSecondAddr = makeAddr("AliceSecondAddr");
		address bob = makeAddr("Bob"); //Bob will try to flag Alice's short 
		address randomUser = makeAddr("randomUser"); //regular user who created a bid order
		
		//A random user create a bid, Alice create a short, which will match with the user's bid
		fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, randomUser);
		fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, alice);
        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT.mulU88(0.5 ether), receiver);
		//Alice then mint the NFT associated to the SR so that it can be transfered
		vm.prank(alice);
		diamond.mintNFT(asset, C.SHORT_STARTING_ID, 0);

		//ETH price drops from 4000 to 2666, making Alice's short liquidatable
		setETH(2666 ether);
		
		// Alice sees Bob attempt to liquidate her short, so she front-run him and transfer the SR
		vm.prank(alice);
		diamond.transferFrom(alice, aliceSecondAddr, 1);
		
		//Bob's attempt revert because the transfer of the short by Alice change the short status to SR.Cancelled
		vm.prank(bob);
		vm.expectRevert(Errors.InvalidShortId.selector);
		diamond.liquidate(asset, alice, C.SHORT_STARTING_ID, shortHintArrayStorage, 0);
	}	
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Do not allow a short owner to transfer its SR if its CR is below the required value.