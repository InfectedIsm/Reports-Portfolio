## NOTE-01 - `random()` is flawed and can be manipulated while being used by `Entropy::requestV2()` to generate user's random numbers
https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L1081-L1091
```solidity
File: contracts/entropy/Entropy.sol
1081:     function random() internal returns (bytes32) {
1082:         _state.seed = keccak256(
1083:             abi.encodePacked(
1084:                 block.timestamp,
1085:                 block.difficulty,
1086:                 msg.sender,
1087:                 _state.seed
1088:             )
1089:         );
1090:         return _state.seed;
1091:     }
```
Generating numbers solely based on the chain's internal state gives validators control over the generated number outcome.  
If some actors communicate (e.g validator and provider), they can take advantage of this to manipulate the odds.

## NOTE-02 - `request()` and `requestV2()` allow Providers to manipulate the outcome by participating themselves as a player

First, `request()` allow users to provide their own random number.
If a protocol integrate with Pyth Entropy, a provider could select a random number with a 100% change to win.
For this, the provider needs to know the hash chain in advance (which it does) and be able to position the request with its own random number (as a user) at the right `sequenceNumber` (which he can with flashbots)
```solidity
File: contracts/entropy/Entropy.sol
334:     function request(
335:         address provider,
336:         bytes32 userCommitment,
337:         bool useBlockHash
338:     ) public payable override returns (uint64 assignedSequenceNumber) {
339:         EntropyStructsV2.Request storage req = requestHelper(
340:             provider,
341:             userCommitment,
342:             useBlockHash,
343:             false,
344:             0
345:         );
346:         assignedSequenceNumber = req.sequenceNumber;
347:         emit Requested(EntropyStructConverter.toV1Request(req));
348:     }
```

Second, even with `requestV2()` functions which uses the `random()` function (see below). To do so:
- the provider can place its tx as the first one for a new block. That way, he already knows the value of `_state.seed`, `block.difficulty` and `block.timestamp`
- he can then brute-force wallet addresses to get the random number he desires that will give a winning outcome when revealed.

```solidity
File: contracts/entropy/Entropy.sol
317:     function requestV2(
318:         address provider,
319:         uint32 gasLimit
320:     ) external payable override returns (uint64 assignedSequenceNumber) {
321:         assignedSequenceNumber = requestV2(provider, random(), gasLimit);
322:     }
...:
...:   // ----- some code ----- //
...:
1081:     function random() internal returns (bytes32) {
1082:         _state.seed = keccak256(
1083:             abi.encodePacked(
1084:                 block.timestamp,
1085:                 block.difficulty,
1086:                 msg.sender,
1087:                 _state.seed
1088:             )
1089:         );
1090:         return _state.seed;
1091:     }
```


## NOTE-03 - using `blockhash` on some chains means the request will become unusable very fast
During the reveal phase, if `blockhash` [was configured to be used](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L275-L275), the contract will ensure only valid `blockhash` are allowed (i.e != `0x0`):
https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L423-L436
```solidity
File: contracts/entropy/Entropy.sol
423:         if (req.useBlockhash) {
424:             bytes32 _blockHash = blockhash(req.blockNumber);
425: 
426:             // The `blockhash` function will return zero if the req.blockNumber is equal to the current
427:             // block number, or if it is not within the 256 most recent blocks. This allows the user to
428:             // select between two random numbers by executing the reveal function in the same block as the
429:             // request, or after 256 blocks. This gives each user two chances to get a favorable result on
430:             // each request.
431:             // Revert this transaction for when the blockHash is 0;
432:             if (_blockHash == bytes32(uint256(0)))
433:                 revert EntropyErrors.BlockhashUnavailable();
434: 
435:             blockHash = _blockHash;
436:         }
437: 
```
`blockhash(block.number)` returns `0` if its more than 256 block old.  
It is interesting to note that some L2 produce blocks with a rate > 1/sec (e.g zkSync where `block.number` returns the number of L2 blocks).  
This means that the requests will become obsolete faster, and impossible to reveal as the call will always `revert`.  

## NOTE-04 - Discreprancy in natspec and implementation: the callback will still be called when requestor is EOA when `req.gasLimit10k != 0`
`revealWithCallback` natspec states that that if requestor is EOA callback will not be called:
```solidity
File: contracts/entropy/Entropy.sol
544:     // Fulfill a request for a random number. This method validates the provided userRandomness
545:     // and provider's revelation against the corresponding commitment in the in-flight request. If both values are validated
546:     // and the requestor address is a contract address, this function calls the requester's entropyCallback method with the
547:     // sequence number, provider address and the random number as arguments. Else if the requestor is an EOA, it won't call it.
...:
...:
554:     function revealWithCallback(
```
While[ its true for the `else` case](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L692-L704), its not true for the [`if(req.gasLimit10k != 0)` case](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L586-L608).  
This does not seems to represent an issue though as for the later, the callback is called through a low level call, which will return successfully if the target is an EOA, which match the `else` case behavior.
(I still need to check into EIP-7702 to see if this changes anything).

## NOTE-05 in `revealWithCallback` the `req.gasLimit10k != 0` do not implement the CEI pattern
While this does not seems to create risks, the [external call](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L594-L594) (Interaction) is done [before clearing the request](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/contracts/contracts/entropy/Entropy.sol#L630-L630) (Effect).

## NOTE-06 Consider locking provider fees until reveal (from [ToB audit](https://github.com/pyth-network/audit-reports/blob/main/2024_01_23/Pyth%20Data%20Association%20-%20Entropy%20-%20Comprehensive%20Report.pdf) january 2024)
ToB ID-12 recommendation is not implemented, but if we consider provider as trusted actors then this is an accepted risk for the user.  
In the current implementation, provider earn the fees when the request is created, and might never reveal the random number to the user(s).
Also, doing that would allow users to recover their fees if the request is never revealed
