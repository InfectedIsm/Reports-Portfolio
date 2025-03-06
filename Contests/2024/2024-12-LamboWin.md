# Lambowin - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    -  [H-01. `createLaunchPad` can be DoS'd by front-running the UniswapV2 pair deployment](#H-01)
    -  [H-02. The `_BUY_MASK` and `_SELL_MASK` might be incorrect depending on the address of vETH once deployed](#H-02)
    -  [H-03. `VirtualToken::cashIn` is malfunctioning when `underlyingToken != LaunchPadUtils.NATIVE_TOKEN`](#H-03)
- ## Medium Risk Findings
    -  [M-01. `LamboRebalanceOnUniswap::_getTokenInOut` formula used to compute rebalancing amount is wrong for a UniV3 pool](#M-01)
    -  [M-02. `createLaunchPad` can be DoS'd by deploying launchpads with `virtualLiquidityAmount = MAX_LOAN_PER_BLOCK` at the start of every blocks](#M-02)
    -  [M-03. `sellQuote` and `buyQuote` are missing deadline check in `LamboVEthRouter`](#M-03)
- ## Low Risk Findings
    -  [L-01. No check on `quoteToken` in `LamboVEthRouter`](#L-01)
    -  [L-02. `VirtualToken` accounting might break if `USDT` or `USDC` fees get activated](#L-02)


# HIGH

## 🔴 H01 - `createLaunchPad` can be DoS'd by front-running the UniswapV2 pair deployment
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboFactory.sol#L72-L72
### Summary
An attacker can scan the mempool and front-run the deployment of UniswapV2 pairs to be deployed, causing a [revert](https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboFactory.sol#L72-L72) in the `LamboFactory::createLaunchPad` as [the pair already exist](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Factory.sol#L27).
### Vulnerability details
`LamboFactory::createLaunchPad` deploy a LamboToken using the OpenZeppelin Clone library, which deploy a proxy with the CREATE opcode.
It is relatively easy to [pre-compute the future deployment address](https://docs.openzeppelin.com/cli/2.8/deploying-with-create2#create) of the LamboToken as it only depends on the sender's (`LamboFactory`) address and nonce.
Once the attacker computed the address, he can call (before the victim) `UniswapV2Factory.createPair(vETH, quoteToken)` to create the pair. 

The attacker can even precompute all future deployed addresses and apply the same procedure without the need of a front-run, making it even easier to pull out.
Once the attacker tx is executed, the victim tx will execute, but once it reaches [the pair creation](https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboFactory.sol#L72-L72) the call will revert, as [the pair already exist](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Factory.sol#L27)

The worst thing here is that because contract nonce only increment with contract deployment, being able to pull it out only once will completely DoS the launchpad creation.
### Impact
High, as:
1) It is possible to completely DoS launchpad creation
2) Once a launchpad creation has been DoS'd, it is not possible anymore to fund that launchpad with vETH using a loan

### Proof of Concept
You can add this test to `test/GeneralTest.t.sol` :
```solidity
	import "../src/interfaces/Uniswap/IPoolFactory.sol";

	// compute the CREATE address based on deployer address and nonce
    function getContractCREATEAddress(address deployer, uint256 nonce) public pure returns (address) {
        return address(uint160(uint256(keccak256(abi.encodePacked(
            bytes1(0xd6),
            bytes1(0x94),
            deployer,
            bytes1(uint8(nonce))
        )))));
    }

    function test_frontrun_launchpad_creation1() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        uint256 MAX_LOAN_PER_BLOCK = 300 ether;

        //method 1: 
        // it is possible to get the nonce of the factory and front-run the pool creation by deploying it first
        // no need to deploy a contract at this address as Uniswap Factory will create the pair even if the address has no code
        uint64 factoryNonce = vm.getNonce(address(factory));
        address conflictingToken = getContractCREATEAddress(address(factory), factoryNonce);
        IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(address(vETH), conflictingToken);

        vm.expectRevert("UniswapV2: PAIR_EXISTS");
        vm.prank(alice);
        factory.createLaunchPad("AliceToken", "ECILAMBO", MAX_LOAN_PER_BLOCK, address(vETH));

        // because deployer nonce only increase for each contract creation, the launchpad creation is completely frozen now
        // as this means all successive tentative will try to create a token at the same address each time
        vm.expectRevert("UniswapV2: PAIR_EXISTS");
        vm.prank(bob);
        factory.createLaunchPad("BobToken", "LAMBOB", MAX_LOAN_PER_BLOCK, address(vETH));

        //method 2: 
        // if we're not capable of front-running, we can simply pre-compute multiple addresses with incremented nonce
        // so that we're sure one of them will match the nonce of the victim tx launchpad creation  
        for(uint i; i < 10; i++) {
            conflictingToken = getContractCREATEAddress(address(factory), factoryNonce + 1 + i);
            IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(address(vETH), conflictingToken);
        }

        vm.expectRevert("UniswapV2: PAIR_EXISTS");
        vm.prank(alice);
        factory.createLaunchPad("LamboToken", "LAMBO", MAX_LOAN_PER_BLOCK, address(vETH));
    }
```
### Recommended Mitigation Steps
Check first if the pair exist by calling [`UniswapV2Factory::getPair()`](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Factory.sol#L10), and only call `createPair` if it does not exist yet.
Although, care must be taken not to allow vETH loans to be taken twice (or more) for a same pair.
One way to do this might be to register every deployed launchpad in a mapping, and only deploy capital to the pool if it hasn't been already done before.

in the below example only a boolean is registered, but you might prefer to register the pair address or more data.
You might otherwise use the `whitelist` mapping of the `VirtualToken` as a registration rather than another mapping.

```diff
    function createLaunchPad(
        string memory name,
        string memory tickname,
        uint256 virtualLiquidityAmount,
        address virtualLiquidityToken
    ) public onlyWhiteListed(virtualLiquidityToken) nonReentrant returns (address quoteToken, address pool) {
        quoteToken = _deployLamboToken(name, tickname);
+       require(deployedLaunchpad[virtualLiquidityToken][quoteToken] == false, "launchpad already deployed");
+		if(IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).getPair() == 0) {
+			deployedLaunchpad[virtualLiquidityToken][quoteToken] = true;
	        pool = IPoolFactory(LaunchPadUtils.UNISWAP_POOL_FACTORY_).createPair(virtualLiquidityToken, quoteToken);
+		}
        VirtualToken(virtualLiquidityToken).takeLoan(pool, virtualLiquidityAmount);
        IERC20(quoteToken).safeTransfer(pool, LaunchPadUtils.TOTAL_AMOUNT_OF_QUOTE_TOKEN);

        // Directly minting to address(0) will cause Dexscreener to not display LP being burned
        // So, we have to mint to address(this), then send it to address(0).
        IPool(pool).mint(address(this));
        IERC20(pool).safeTransfer(address(0), IERC20(pool).balanceOf(address(this)));        
        emit PoolCreated(virtualLiquidityToken, quoteToken, pool, virtualLiquidityAmount);
    }
```

____________________________________


## 🔴H02 - the `_BUY_MASK` and `_SELL_MASK` might be incorrect depending on the address of vETH once deployed
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/rebalance/LamboRebalanceOnUniwap.sol#L27-L28
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/rebalance/LamboRebalanceOnUniwap.sol#L165-L165
### Summary
The `LamboRebalanceOnUniwap` contract, by having its `_BUY_MASK = 1 << 255` assume the Uniswap V3 pool values of  `token0` and `token1`, while these values will depend on the generated address of the `vETH` token.
This means there's a chance that `_BUY_MASK` will in fact correspond to a sell direction for the swap rather than a buy.

### Vulnerability details
`UniswapV3Pool.sol` has a [low-level `swap` function](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolActions#swap) which have a `bool zeroForOne` parameter to indicate the direction of the swap.
This is that function that is used by the `OKXRouter` used in `LamboRebalanceOnUniwap.sol` to swap vETH and WETH in the UniswapV3 pool:
1) the `_executeBuy` and `_executeSell` both calls [`OKXRouter::uniswapV3SwapTo`](https://github.com/okx/WEB3-DEX-OPENSOURCE/blob/93fbaa3a7a1c73ed11f0750c5d56627c45f9fd00/contracts/8/DexRouter.sol#L624-L632)
2) `OKXRouter::uniswapV3SwapTo` calls an internal function [`_uniswapV3Swap`](https://github.com/okx/WEB3-DEX-OPENSOURCE/blob/93fbaa3a7a1c73ed11f0750c5d56627c45f9fd00/contracts/8/UnxswapV3Router.sol#L48), which unfortunately is written in assembly
But we can see in this file: 
- [L32-33](https://github.com/okx/WEB3-DEX-OPENSOURCE/blob/93fbaa3a7a1c73ed11f0750c5d56627c45f9fd00/contracts/8/UnxswapV3Router.sol#L32-33): `SWAP_SELECTOR = 0x128acb080...` which is the function selector of `swap(address,bool,int256,uint160,bytes)`
- [L79-114](https://github.com/okx/WEB3-DEX-OPENSOURCE/blob/93fbaa3a7a1c73ed11f0750c5d56627c45f9fd00/contracts/8/UnxswapV3Router.sol#L82-L114): that indeed the mask is used to either call `swap` with `zeroForOne` equal to `true` or `false`

If this boolean is equal to `true`, this means that the direction is to sell `token0` for `token1` (and the opposite for `false`).

But in a `vETH/WETH` pool, how can one know which one will be `token0` and `token1`?
This is managed at pool creation, by simply comparing the numerical value of both address, the lowest value being assigned to `token0` (and the other one to `token1`)

This means `_BUY_MASK` and `_SELL_MASK` must be set at construction by checking first the `token0` and `token1` values.

### Impact
Inverse behavior of the `LamboRebalanceOnUniwap::rebalance`, buying instead of selling (and vice-versa) 

### Recommended Mitigation Steps
Compare `token0` and `token1` of the `uniswapPool` and set the masks accordingly.

____________________________________


## 🔴H03 - `VirtualToken::cashIn` is malfunctioning when `underlyingToken != LaunchPadUtils.NATIVE_TOKEN`
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/VirtualToken.sol#L78-L78
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/VirtualToken.sol#L124-L130
### Summary
While the `cashIn` function seems at first glance to be capable of managing both ETH and ERC20, it is not the case, and if it is called with an `underlyingToken` being an ERC20, tokens will be transfered from the `msg.sender`, but nothing will be minted to `msg.sender` in return.
### Vulnerability details
A `VirtualToken` is an ERC20 token capable of wrapping any token (ETH or an ERC20) to its "virtual" counter part at a 1:1 rate, as much unwrapping it.
Each `VirtualToken` have an `underlyingToken` state variable indicating which token is wrapped inside.
The `cashIn(uint256 amount)` function allow the caller to deposit `amount` of that `underlyingToken`, and get minted the "virtual" counterpart in exchange.
But as we can see in the code snippet below, the `amount` parameter is only used once, when the `underlyingToken` is ETH, ensuring enough ETH has been sent with the call.
While this cause no issue in this case, as `msg.value == amount` gets minted to the caller, this pose a huge issue when `underlyingToken != NATIVE_TOKEN`
Indeed, when `underlyingToken` is an ERC20, `msg.value == 0`, as the token gets transfered from the caller to the `VirtualToken`
But then, the minting process is wrong, as the caller will send tokens, but get minted 0 virtual assets.

```solidity
File: src/VirtualToken.sol
72:     function cashIn(uint256 amount) external payable onlyWhiteListed {
73:         if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
74:             require(msg.value == amount, "Invalid ETH amount");
75:         } else {
76:             _transferAssetFromUser(amount);
77:         }
78:         _mint(msg.sender, msg.value);			//❌ `msg.value` should be `amount`
79:         emit CashIn(msg.sender, msg.value);
80:     }
...:			//* --------  some code -------- *//
...:
124:     function _transferAssetFromUser(uint256 amount) internal {
125:         if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
126:             require(msg.value >= amount, "Invalid ETH amount");
127:         } else {
128:             IERC20(underlyingToken).safeTransferFrom(msg.sender, address(this), amount);
129:         }
130:     }
```

Also, less impactful, we see that the logic is duplicated, as the checks performed by `_transferAssetFromUser` are exactly the same as in the `cashIn`
For this reason, the `if` statement in `cashIn` can be removed, and calling `_transferAssetFromUser` should be enough.
### Impact
Loss of funds
### Recommended Mitigation Steps
Both the minting issue and duplicated logics are mitigated here:

```diff
    function cashIn(uint256 amount) external payable onlyWhiteListed {
-       if (underlyingToken == LaunchPadUtils.NATIVE_TOKEN) {
-           require(msg.value == amount, "Invalid ETH amount");
-       } else {
           _transferAssetFromUser(amount);
-       }
-        _mint(msg.sender, msg.value);
+        _mint(msg.sender, amount);                 
        emit CashIn(msg.sender, msg.value);

    }
```

# MEDIUM



## 🟡 M01 - `LamboRebalanceOnUniswap::_getTokenInOut` formula used to compute rebalancing amount is wrong for a UniV3 pool

### Summary
The formula implemented assume that the pool is based on a constant sum AMM formula (`x+y = k`), and also elude the fact that reserves in a UniV3 pool do no directly relate to the price because of the 1-sided ticks liquidity.
This make the function imprecise at best, and highly imprecise when liquidity is deposited in distant ticks, with no risk involved for actors depositing in those ticks. 

### Vulnerability details
The `previewRebalance` function has been developed to output all the necessary input parameters required to call the `rebalance` function, which goal is to swap tokens in order to keep the peg of the virtual token in comparison to its counterpart (e.g keep vETH/ETH prices = 1):

```solidity
File: src/rebalance/LamboRebalanceOnUniwap.sol
128:     function _getTokenBalances() internal view returns (uint256 wethBalance, uint256 vethBalance) {
129:         wethBalance = IERC20(weth).balanceOf(uniswapPool);         <<❌(1) this does not represent the active tick
130:         vethBalance = IERC20(veth).balanceOf(uniswapPool);
131:     }
132: 
133:     function _getTokenInOut() internal view returns (address tokenIn, address tokenOut, uint256 amountIn) {
134:         (uint256 wethBalance, uint256 vethBalance) = _getTokenBalances();
135:         uint256 targetBalance = (wethBalance + vethBalance) / 2;    <<❌(2) wrong formula
136:
137:         if (vethBalance > targetBalance) {
138:             amountIn = vethBalance - targetBalance;
139:             tokenIn = weth;
140:             tokenOut = veth;
141:         } else {
142:             amountIn = wethBalance - targetBalance;
143:             tokenIn = veth;
144:             tokenOut = weth;
145:         }
146: 
147:         require(amountIn > 0, "amountIn must be greater than zero");
148:     }
149: 
```

The implemented formula is incorrect, as it will not rebalance the pool for 2 reasons:
1) In Uniswap V3, LPs can deposit tokens in any ticks they want, even though those ticks are not active and do not participate to the actual price.
   But those tokens will be held by the pool, and thus be measured by `_getTokenBalances` 
2) The formula used to compute the targetBalance is incorrect because of how works the constant product formula `x*y=k`

Regarding (2), consider this situation:
	WETH balance: 1000
	vETH balance: 900
	`targetBalance = (1000 + 900) / 2 = 950`
	 `amountIn = 1000 - 950 = 50 (vETH)`

Swapping 50 vETH into the pool will not return 50 WETH because of the inherent slippage of the constant product formula.

Now, add to this bullet (1), and the measured balance will be wrong anyway because of the liquidity deposited in inactive ticks, making the result even more shifted from the optimal result.
### Impact
The function is not performing as intended, leading to invalid results which complicate the computation of rebalancing amounts necessary to maintain the peg.  
Since this function is important to maintain the health of vETH as it has access to on-chain values, allowing precise rebalancing, failing to devise and implement a reliable solution for rebalancing before launch could result in significant issues. 

### Tools Used
Manual review

### Recommended Mitigation Steps
Reconsider the computations of rebalancing amounts for a more precise one if keeping a 1:1 peg is important.
You might want to get inspiration from [`USSDRebalancer::rebalance()`](https://github.com/USSDofficial/ussd-contracts/blob/b16d1d2a8ffe2725d5950fec5ac55dc7339994b7/contracts/USSDRebalancer.sol#L125-L150)


____________________________________


## 🟡 M02 -  `createLaunchPad` can be DoS'd by deploying launchpads with `virtualLiquidityAmount = MAX_LOAN_PER_BLOCK` at the start of every blocks
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/VirtualToken.sol#L93-L93
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboFactory.sol#L74-L74
### Summary & Vulnerability details
A `VirtualToken` allow anyone, through the factory, to deploy virtual liquidity to an Uniswap pair when creating a launchpad.
But the quantity of available liquidity to loan per block is limited by `MAX_LOAN_PER_BLOCK`:

```solidity
File: src/VirtualToken.sol
088:     function takeLoan(address to, uint256 amount) external payable onlyValidFactory {
089:         if (block.number > lastLoanBlock) {
090:             lastLoanBlock = block.number;
091:             loanedAmountThisBlock = 0;
092:         }
093:         require(loanedAmountThisBlock + amount <= MAX_LOAN_PER_BLOCK, "Loan limit per block exceeded");
094: 
095:         loanedAmountThisBlock += amount;
096:         _mint(to, amount);
097:         _increaseDebt(to, amount);
098: 
099:         emit LoanTaken(to, amount);
100:     }
```

There no cost other than gas to deploy a launchpad, and there is a limit on the amount of virtual liquidity that can be deployed every blocks. 
For this reason, it is possible for an attacker to front-run in each blocks every first call to `createLaunchPad` with another `createLaunchPad` call consuming all the available 
### Impact
Temporary DoS (than can be repeated indefinitely) of a critical function of the protocol
### Proof of Concept
You can add this test to `test/GeneralTest.t.sol` :
```solidity
    function test_frontrun_launchpad_creation2() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        uint256 MAX_LOAN_PER_BLOCK = 300 ether;

        // alice create a launchpad at the beginning of the block for no cost other than gas
        vm.prank(alice);
        factory.createLaunchPad("LamboToken", "LAMBO", MAX_LOAN_PER_BLOCK, address(vETH));

        // bob or anyone cannot create launchpad until next block
        // this can be repeated each blocks
        vm.expectRevert("Loan limit per block exceeded");
        vm.prank(bob);
        factory.createLaunchPad("LamboToken", "LAMBO", 10 ether, address(vETH));
    }
```

### Recommended Mitigation Steps
Add a fee taken from every users creating launchpad, and limit each launchpad creation to a fraction of the block limit liquidity.

____________________________________


## 🟡 M03 - `sellQuote` and `buyQuote` are missing deadline check in `LamboVEthRouter` 
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboVEthRouter.sol#L102-L102
https://github.com/code-423n4/2024-12-lambowin/blob/main/src/LamboVEthRouter.sol#L148-L148
### Summary

`sellQuote` and `buyQuote` are missing deadline check in `LamboVEthRouter`.
Because of that, users transactions could remain in the mempool for an extended time, and be executed at an unfavorable price at the time of the execution.
### Vulnerability details

The protocol has made the choice to develop its own router to swap tokens for users, which imply calling the low level [`UniswapV2Pair::swap`](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L158) function:
```solidity
// this low-level function should be called from a contract which performs important safety checks
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external lock {
	require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
	(uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
	require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');
```
As the comment indicate, this function require important safety checks to be performed.

A good example of safe implementation of such call can be found in the [`UniswapV2Router02::swapExactTokensForTokens`](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L224) function:
```solidity
function swapExactTokensForTokens(
	uint amountIn,
	uint amountOutMin,
	address[] calldata path,
	address to,
	uint deadline
) external virtual override ensure(deadline) returns (uint[] memory amounts) {
	amounts = UniswapV2Library.getAmountsOut(factory, amountIn, path);
	require(amounts[amounts.length - 1] >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT');
	TransferHelper.safeTransferFrom(
		path[0], msg.sender, UniswapV2Library.pairFor(factory, path[0], path[1]), amounts[0]
	);
	_swap(amounts, path, to);
}
```

As we can see, 2 safety parameters are present here: `amountOutMin` and `deadline`
Now, if we look at `SellQuote` (`buyQuote` having the same issue):
```solidity
File: src/LamboVEthRouter.sol
148:     function _buyQuote(address quoteToken, uint256 amountXIn, uint256 minReturn) internal returns (uint256 amountYOut) {
149:         require(msg.value >= amountXIn, "Insufficient msg.value");
150: 
...:
...:       //* ---------- some code ---------- *//
...:
168:         require(amountYOut >= minReturn, "Insufficient output amount");                                     
```

We can see that no `deadline` parameter is present.
### Impact
Users transactions could remain in the mempool for an extended time, and be executed at an unfavorable price at the time of the execution.

### Recommended Mitigation Steps
Add a deadline parameter

____________________________________

# Low & QA

## 🔵 L-01 - No check on `quoteToken` in `LamboVEthRouter`

### Vulnerability details
Any `quoteToken` can be provided to `LamboVEthRouter::sellQuote` and `LamboVEthRouter::buyQuote`, while there doesn't seem to be any risk involved with unwhitelisted tokens right now, new vulnerabilities might appear through the evolution of the protocol,
### Impact
Allowing anyone to call unverified tokens on the router behalf might be dangerous in future upgrades.
### Recommended Mitigation Steps
Check the provided `quoteToken` against a list of all launchpad that were deployed by the factory.


____________________________________
## 🔵 L-02 - `VirtualToken` accounting might break if `USDT` or `USDC` fees get activated

### Finding description and impact

While USDC and USDT do not have fee on transfer today, a fee can be activated at anytime.
The issue is that `cashIn` and `cashOut` functions rely on the `amount` parameter to decide how much virtual counterpart to mint/burn to/from `msg.sender`.
Doing so will break accounting.
### Proof of Concept

As a simple scenario:
1. USDT fees are activated to 5%
2. 100 USDT are deposited (`cashIn`), in response 100 vUSDT get minted
3. Because of fees the `VirtualToken` only received 95 USDT
4. It is not possible to withdraw 100 USDT as only 95 USDT are available 
5. 95 USDT are withdrawn, now there are 5 vUSDT left in `VirtualToken`
### Recommended mitigation steps

Measure how much token have been received and mint the same quantity of virtual tokens.

