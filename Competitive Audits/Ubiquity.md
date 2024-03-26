
____________________________________
# [H-01] - TWAP oracle is manipulable as the only time-window requirement is `blockTimestamp - ts.pricesBlockTimestampLast > 0` 

## Vulnerability details
TWAP oracle are used to make it exponentially expensive to manipulate the price. 
This is done by not getting the spot price of a pool, but rather the time weighted averaged price over a period of time.
The longer the period, the harder it is to manipulate the price (but the laggier is the price representation)

The `update` function in `LibTWAPOracle` can be called once every block as the only requirement is the time difference between to consecutive calls is >0.
This means the price is calculated between block N and block N-1, which really reduce the impact of the time weighing here.

This already happened 

## Impact
The TWAP advantage is not used correctly, increasing the feasibility of an attack based on Dollar price manipulation.
The `LibUbiqityPool.getDollarPriceUsd()` make use of these prices [here](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L347) and [here](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L419), which is used as a protection mechanism for the Dollar peg.
We also can imagine in the future that this TWAP oracle can be used for other functions, which will then be vulnerable to a price manipulation.

## Proof of Concept

```solidity
File: src\dollar\libraries\LibTWAPOracle.sol
69:     function update() internal {
70:         TWAPOracleStorage storage ts = twapOracleStorage(); 
71:         (
72:             uint256[2] memory priceCumulative,
73:             uint256 blockTimestamp
74:         ) = currentCumulativePrices();
75: @>      if (blockTimestamp - ts.pricesBlockTimestampLast > 0) {	
76:             // get the balances between now and the last price cumulative snapshot
77:             uint256[2] memory twapBalances = IMetaPool(ts.pool)			
78:                 .get_twap_balances(				
79:                     ts.priceCumulativeLast,
80:                     priceCumulative,
81: @>                  blockTimestamp - ts.pricesBlockTimestampLast	
82:                 );													
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Use a configurable time period, and either update the TWAP price once very period, or use a sliding window.
Uniswap propose [examples here](https://github.com/Uniswap/v2-periphery/tree/master/contracts/examples)




____________________________________
# [M-01] - AmoMinterBorrow can brick the pool by causing an underflow state 
### (non reward but fixed)

## Vulnerability details

The `amoMinterBorrow` function in the UbiquityPool contract can lead to a underflow state in the pool's collateral balance. 

This function allows an AMO minter to borrow collateral from the pool without accounting for the `unclaimedPoolCollateral`, which is increased everytime a user redeem Dollar tokens. 

This can create a scenario where the total collateral balance `collateral.balanceOf(address(this))` becomes less than the `unclaimedPoolCollateral`. 
This discrepancy leads to an arithmetic underflow in functions like `redeemDollar`, `mintDollar`, and `collateralUsdBalance`.

```solidity
File: src\dollar\libraries\LibUbiquityPool.sol
437:         // checks
438:         require(
439:             collateralOut <=
440:                 (IERC20(poolStorage.collateralAddresses[collateralIndex]))
441:                     .balanceOf(address(this))
442:                     .sub(poolStorage.unclaimedPoolCollateral[collateralIndex]),
443:             "Insufficient pool collateral"
444:         );
```

## Impact
Once this underflow state is entered, there's 2 ways to exit it:
1) directly transfer collateral tokens to the contract to make its `balance > unclaimedPoolCollateral`
2) Users who redeemed call `collectRedemption` until `unclaimedPoolCollateral < balance`. Until then, its not possible to mint or redeem.

## Proof of Concept

Add this test to UbuiquityPoolFacet.t.sol

```solidity
	function test_auditAmoBrickPool() public {
		address alice = makeAddr("alice");
		uint256 collateralIndex = 0;
        // vm.prank(address(dollarAmoMinter));

		//setting read-convenient configuration
		vm.startPrank(admin);
        ubiquityPoolFacet.setPriceThresholds(
            1000000, // mint threshold
            1000000 // redeem threshold
        );

		ubiquityPoolFacet.setFees(
            0, // collateral index
            0, // 1% mint fee
            0 // 2% redeem fee
        );
		vm.stopPrank();

		// balance setup of users
		uint256 depositAlice = 1000;
		collateralToken.mint(alice, depositAlice);
		vm.prank(alice);
        collateralToken.approve(address(ubiquityPoolFacet), type(uint).max);

		//Alice borrows, she sends collateral to the  which increase the collateral balance of the pool
        vm.prank(alice);
        (uint256 totalDollarMintAlice, /*uint256 collateralNeeded*/) = 
			ubiquityPoolFacet.mintDollar(collateralIndex, depositAlice, 0, type(uint).max);

		//Alice redeem its Dollars, which increases the unclaimedPoolCollateral
        vm.prank(alice);
		ubiquityPoolFacet.redeemDollar(collateralIndex, totalDollarMintAlice, 0);

		//AmoMinter borrow some collateral, which decrease the collateral balance of the pool
		console.log("------ AmoMinter borow ------");
		vm.prank(address(dollarAmoMinter));
		ubiquityPoolFacet.amoMinterBorrow(1);

		//this set a state where collateral.balanceOf(address(this)) < (poolStorage.unclaimedPoolCollateral[collateralIndex])
		//creating a situation of underflow in redeemCollateral, mintDollar and collateralUsdBalance

		bytes memory arithmeticError = abi.encodeWithSignature("Panic(uint256)", 0x11);

		vm.expectRevert(arithmeticError);
        vm.prank(alice);
		ubiquityPoolFacet.redeemDollar(collateralIndex, 0, 0);

        vm.expectRevert(arithmeticError);
		vm.prank(alice);
		ubiquityPoolFacet.mintDollar(collateralIndex, depositAlice, 0, type(uint).max);

        vm.expectRevert(arithmeticError);
		ubiquityPoolFacet.collateralUsdBalance();
	}
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Make sure AmoMinter can only borrow amounts up to `balance - unclaimedPoolCollateral`


____________________________________
# [M-02] - LibTWAPOracle.setPool can be DoS'd by sending 1 wei to imbalance pools reserves
### (non reward but fixed)

## Vulnerability details
the `setPool` function is used to set a Curve MetaPool as the TWAP oracle reference. 
There's multiple sanity checks performed inside, one of them ensuring that the reserve of both tokens is the same.
It make the call vulnerable to DoS front-running by sending 1 wei of any of both tokens to imbalance the pool and make the call revert.
Also, if you check the different MetaPools, you'll see that reserves aren't balanced.

## Impact
Not allowing to set up the oracle

## Proof of Concept

```solidity
File: src\dollar\libraries\LibTWAPOracle.sol
32:     function setPool(address _pool, address _curve3CRVToken1) internal {
33:         require(
34:             IMetaPool(_pool).coins(0) ==
35:                 LibAppStorage.appStorage().dollarTokenAddress,
36:             "TWAPOracle: FIRST_COIN_NOT_DOLLAR"
37:         );
38:         TWAPOracleStorage storage ts = twapOracleStorage();
39: 
40:         // coin at index 0 is Ubiquity Dollar and index 1 is 3CRV
41:         require(
42:             IMetaPool(_pool).coins(1) == _curve3CRVToken1,
43:             "TWAPOracle: COIN_ORDER_MISMATCH"
44:         );
45: 
46:         uint256 _reserve0 = uint112(IMetaPool(_pool).balances(0));
47:         uint256 _reserve1 = uint112(IMetaPool(_pool).balances(1));
48:
49:         // ensure that there's liquidity in the pair
50:         require(_reserve0 != 0 && _reserve1 != 0, "TWAPOracle: NO_RESERVES");
51:         // ensure that pair balance is perfect
52: @>      require(_reserve0 == _reserve1, "TWAPOracle: PAIR_UNBALANCED"); //@audit this can be DoS'd
53:         ts.priceCumulativeLast = IMetaPool(_pool).get_price_cumulative_last()
54:         ts.pricesBlockTimestampLast = IMetaPool(_pool).block_timestamp_last();
55:         ts.pool = _pool;
56:         // dollar token is inside the diamond
57:         ts.token1 = _curve3CRVToken1;
58:         ts.price0Average = 1 ether;
59:         ts.price1Average = 1 ether;
60:     }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
There is no reason to ensure balances are perfect in the pool

____________________________________
# [QA] - Wrong rouding direction in mintDollar make it possible to mint Dollars against 0 collateral

## Vulnerability details
The `getDollarInCollateral(uint256 collateralIndex, uint256 dollarAmount)` function round down the value, and is used in both the `mintDollar` and `redeemDollar` functions.

While its a correct implementation in the `redeemDollar`, it is not for the `mintDollar` as it make it possible for a user to mint Dollar tokens against 0 collateral due to the rounding down of the `collateralNeeded` result.

This can though only happen with collateral with decimals <18, and these kind of tokens seems expected in the future as there's a parameter [dedicated for that](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L56)

## Impact
Leak of value for the pool, that will add up over time.

## Proof of Concept


## Tools Used
Manual review

## Recommended Mitigation Steps
Round up, or ensure that `collateralNeeded > 0` when minting.

