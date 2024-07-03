# M01 - Important chainlink oracle checks missing in WooracleV2_2.sol
## Summary
By not checking sequencer liveliness or price staleness, WooFi own prices are open to manipulation.

## Vulnerability Detail
The protocol relies on chainlink oracle price to ensure WooFi pool price do not deviate too far (which could be the effect of a manipulation)
This is done by [checking woPrice against cloPrice (chainlink price)](https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/wooracle/WooracleV2_2.sol#L250-L251)

In case of down sequencer/stale price, the bound check will be verified on stale data, which will open up possibilities to take advantage of WooFi price and reserves.

## Impact
Incorrect pricing of WooFi assets.

## Code Snippet

No staleness check in `_cloPriceInQuote` 
https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/wooracle/WooracleV2_2.sol#L348-L368
```solidity
File: contracts\wooracle\WooracleV2_2.sol
348:     function _cloPriceInQuote(address _fromToken, address _toToken)
349:         internal
350:         view
351:         returns (uint256 refPrice, uint256 refTimestamp)
352:     {
353:         address baseOracle = clOracles[_fromToken].oracle;
354:         if (baseOracle == address(0)) {
355:             return (0, 0);
356:         }
357:         address quoteOracle = clOracles[_toToken].oracle;
358:         uint8 quoteDecimal = clOracles[_toToken].decimal;
359: 		
360:         (, int256 rawBaseRefPrice, , uint256 baseUpdatedAt, ) = AggregatorV3Interface(baseOracle).latestRoundData();
361:         (, int256 rawQuoteRefPrice, , uint256 quoteUpdatedAt, ) = AggregatorV3Interface(quoteOracle).latestRoundData(); 
362:         uint256 baseRefPrice = uint256(rawBaseRefPrice);																
363:         uint256 quoteRefPrice = uint256(rawQuoteRefPrice);
364: 
365:         // NOTE: Assume wooracle token decimal is same as chainlink token decimal.
366:         uint256 ceoff = uint256(10)**quoteDecimal;
367:         refPrice = (baseRefPrice * ceoff) / quoteRefPrice;
368:         refTimestamp = baseUpdatedAt >= quoteUpdatedAt ? quoteUpdatedAt : baseUpdatedAt;
369:     }
```

`_cloPriceInQuote` returns an incorrect price L247, which then is price bound checked L250-251
https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/wooracle/WooracleV2_2.sol#L250-L251
```solidity
File: contracts\wooracle\WooracleV2_2.sol
243:     function price(address _base) public view override returns (uint256 priceOut, bool feasible) {
244:         uint256 woPrice_ = uint256(infos[_base].price);
245:         uint256 woPriceTimestamp = timestamp;
246: 
247:         (uint256 cloPrice_, ) = _cloPriceInQuote(_base, quoteToken);
248:		
249:         bool woFeasible = woPrice_ != 0 && block.timestamp <= (woPriceTimestamp + staleDuration);
250:         bool woPriceInBound = cloPrice_ == 0 ||
251:             ((cloPrice_ * (1e18 - bound)) / 1e18 <= woPrice_ && woPrice_ <= (cloPrice_ * (1e18 + bound)) / 1e18);
252: 
253:         if (woFeasible) {
254:             priceOut = woPrice_;
255:             feasible = woPriceInBound;
256:         } else {
257:             priceOut = clOracles[_base].cloPreferred ? cloPrice_ : 0;
258:             feasible = priceOut != 0;
259:         }
260:     }
```


## Tool used
Manual Review

## Recommendation
To mitigate this issue, consider integrating an external uptime feed such as [Chainlink's L2 Sequencer Feeds](https://docs.chain.link/data-feeds/l2-sequencer-feeds) and act in consequences.

_____


# M02 - `WooCrossChainRouterV4::_handleERC20Received` do not set back allowance to 0 if swap failed

## Summary
In `WooCrossChainRouterV4::_handleNativeReceived`, swap are wrapped in try/catch blocks, in case of revert, the catch is used reset the allowance back to zero to anticipate [a known issues with USDT](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#approval-race-protections)
This implementation has been forgotten in `_handleERC20Received`

## Vulnerability Detail
In case of swap fail, USDT allowance will not be set back to 0, which will DoS next swap based on USDT (which is an asset [used by WooPPV2 on arbitrum](https://arbiscan.io/address/0xeff23b4be1091b53205e35f3afcd9c7182bf3062#tokentxns))
The fact that it has been implemented in `_handleNativeReceived` confirm it is not the expected behavior in `_handleERC20Received`

## Impact
Revert of the cross-swap on receiving chain, leading to user not receiving their funds.

## Code Snippet
Allowance correctly implemented in `_handleNativeReceived`:
- https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/CrossChain/WooCrossChainRouterV4.sol#L366
- https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/CrossChain/WooCrossChainRouterV4.sol#L331

forgotten in `_handleERC20Received`:
- https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/CrossChain/WooCrossChainRouterV4.sol#L443
- https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/CrossChain/WooCrossChainRouterV4.sol#L478

## Tool used
Manual Review

## Recommendation
Implement same allowance mechanism as `_handleNativeReceived` in `_handleERC20Received`

_____

# M02 - Chainlink oracle fallback protection is ineffective if cloPrice returns 0

## Summary
WooFi oracle `WooracleV2_2` ensure that its prices are not manipulated by bound checking its price with chainlink's prices.
But in case of a returned `price = 0`, bounds are not checked with anything and `priceInBound` is set to `True`


## Vulnerability Detail
A malfunction of Chainlink oracle causing a returned price of 0 would skip price bound checks, opening up doors for price manipulation out of configured bounds.
When `woFeasible` is `True`, `priceOut` will be equal to woPrice_, even though it has not been bound checked with a trusted source. 
This basically defeat the goal of a fallback, and make the oracle vulnerable to woPrice manipulation, as it can deviate from its price with no limits.

## Impact
Price manipulation of WooFi pools.

## Code Snippet

https://github.com/sherlock-audit/2024-03-woofi-swap/blob/main/WooPoolV2/contracts/wooracle/WooracleV2_2.sol#L250-L251
```solidity
File: contracts\wooracle\WooracleV2_2.sol
243:     function price(address _base) public view override returns (uint256 priceOut, bool feasible) {
244:         uint256 woPrice_ = uint256(infos[_base].price);
245:         uint256 woPriceTimestamp = timestamp;
246: 
247:         (uint256 cloPrice_, ) = _cloPriceInQuote(_base, quoteToken);
248:		
249:         bool woFeasible = woPrice_ != 0 && block.timestamp <= (woPriceTimestamp + staleDuration);
250:         bool woPriceInBound = cloPrice_ == 0 ||
251:             ((cloPrice_ * (1e18 - bound)) / 1e18 <= woPrice_ && woPrice_ <= (cloPrice_ * (1e18 + bound)) / 1e18);
252: 
253:         if (woFeasible) {
254:             priceOut = woPrice_;
255:             feasible = woPriceInBound;
256:         } else {
257:             priceOut = clOracles[_base].cloPreferred ? cloPrice_ : 0;
258:             feasible = priceOut != 0;
259:         }
260:     }
```

## Tool used
Manual Review

## Recommendation
Either automatically consider price not in bound, or add other trusted price sources as fallback to keep the WooFi oracle running with up to date prices to check to.

