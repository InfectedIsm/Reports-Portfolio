# C4: Asymmetry (March 2023)

# **Low Risk and Non-Critical Issues**

## **L-01 Adding a _contractAddress not implementing the IDerivative interface will cause a DoS of all operations**

### **Impact**

Adding an address not implementing (or incorrectly implementing) the IDerivative interface (owner could by mistake copy-paste the wrong address, like an EOA or a totally different contract, or forget a function in its new Derivative contract) will cause a DoS of all (or some) operations.

As it can be seen in the code snippet, no verification is done on the added derivative :

```solidity
0File: contracts\SafeEth_orig.sol
182:     function addDerivative(
183:         address _contractAddress,
184:         uint256 _weight
185:     ) external onlyOwner {
186:         derivatives[derivativeCount] = IDerivative(_contractAddress);
187:         weights[derivativeCount] = _weight;
188:         derivativeCount++;
189:
190:         uint256 localTotalWeight = 0;
191:         for (uint256 i = 0; i < derivativeCount; i++)
192:             localTotalWeight += weights[i];
193:         totalWeight = localTotalWeight;
194:         emit DerivativeAdded(_contractAddress, _weight, derivativeCount);
195:     }

```

Impacted functions : `rebalanceToWeights`, `stake`, `unstake`, all of them calling a function from a derivative.

I've qualifed it as Medium because the function of the protocol or its availability could be impacted.

### **Proof of Concept**

This is because calling a function using `Interface(address).function()` from an address which does not implement the function cause a revert.

1)Owner add a new derivative but make a mistake while providing the _contractAddress parameter and provide an address which does not correctly implement the interface (let's say it is an EOA or a completely different contract)

2)the derivative is added to the mapping, and derivativeCount is incremented

3)a user call `stake()`, which calls [L73](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L73-L74) `derivatives[i].balance()`, function does not exists, causing revert

4) Nothing is available in the contract for the owner to revert its mistake

### **Tools Used**

Manual audit

### **Recommended Mitigation Steps**

Consider adding an ERC-165 interface to the Derivatives contracts, and check that the address implements the interface to ensure the added Derivative is compliant. If the address do not implement the interface, the function should revert.If the developer do not want to use ERC-165, he should at least create an additional function to remove or replace a derivative (and update the DerivativeCount accordingly).

## **N-01 Better to use `__ownable_init()` instead of `_transferOwnership()` as this is the recommended way to initialize the Ownable contract.**

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L53

### **SafEth::initialize**

```solidity
File: contracts\SafeEth.sol
48:     function initialize(
49:         string memory _tokenName,
50:         string memory _tokenSymbol
51:     ) external initializer {
52:         ERC20Upgradeable.__ERC20_init(_tokenName, _tokenSymbol);
53:         _transferOwnership(msg.sender);

```

## **N-02 No event triggered when `minAmount` and `maxAmount` are set**

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L54-L55

Deploying a new implementation will change these values, users must be aware of that change

### **SafeEth::initialize**

```solidity
File: contracts\SafeEth.sol
48:     function initialize(
49:         string memory _tokenName,
50:         string memory _tokenSymbol
51:     ) external initializer {
52:         ERC20Upgradeable.__ERC20_init(_tokenName, _tokenSymbol);
53:         _transferOwnership(msg.sender);
54:         minAmount = 5 * 10 ** 17; // initializing with .5 ETH as minimum
55:         maxAmount = 200 * 10 ** 18; // initializing with 200 ETH as maximum
56:     }

```

---

# **Gas Optimizations**

## **G-01 Cache storage value `derivativeCount` in memory before using it in a loop**

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L71Storing `derivativeCount`

### **SafeEth::stake**

```solidity
File: contracts\SafeEth.sol
70:         // Getting underlying value in terms of ETH for each derivative
71:         for (uint i = 0; i < derivativeCount; i++)

```

better to do:

```solidity
        // Getting underlying value in terms of ETH for each derivative
        uint256 _derivativeCount = derivativeCount;
        for (uint i = 0; i < _derivativeCount; i++)

```

Also present here:

```solidity
File: contracts\SafeEth.sol
084:         for (uint i = 0; i < derivativeCount; i++) {
---
113:         for (uint256 i = 0; i < derivativeCount; i++) {
---
140:         for (uint i = 0; i < derivativeCount; i++) {
---
171:         for (uint256 i = 0; i < derivativeCount; i++)
---
191:         for (uint256 i = 0; i < derivativeCount; i++)

```

- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L84
- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L113
- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L140
- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L147
- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L171
- https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L191

## **G-02 x += y costs more gas than x = x + y**

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L72-L74

### **SafeEth::stake**

```solidity
File: contracts\SafeEth.sol
72:             underlyingValue +=
73:                 (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
74:                     derivatives[i].balance()) /
75:                 10 ** 18;

```

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L95

```solidity
File: contracts\SafeEth.sol
95:             totalStakeValueEth += derivativeReceivedEthValue;

```

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L172

### **SafeEth::adjustWeight**

```solidity
File: contracts\SafeEth.sol
172:             localTotalWeight += weights[i];

```

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L192

### **SafeEth::addDerivative**

```solidity
File: contracts\SafeEth.sol
192:             localTotalWeight += weights[i];

```

## **G-03 Two for loops could be combined into one to save gas**

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L63-L101The second loop execution do not depends either on `underlyingValue` nor `preDepositPrice`The calculation of `underlyingValue` could be done inside the second loop, and `predepositPrice` calculated after the loop. This would save one loop but also access to the `derivatives` mapping

```solidity
File: contracts\SafeEth.sol
063:     function stake() external payable {
064:         require(pauseStaking == false, "staking is paused");
065:         require(msg.value >= minAmount, "amount too low");
066:         require(msg.value <= maxAmount, "amount too high");
067:
068:         uint256 underlyingValue = 0;
069:
070:         // Getting underlying value in terms of ETH for each derivative
071:         for (uint i = 0; i < derivativeCount; i++)
072:             underlyingValue +=
073:                 (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
074:                     derivatives[i].balance()) /
075:                 10 ** 18;
076:
077:         uint256 totalSupply = totalSupply();
078:         uint256 preDepositPrice; // Price of safETH in regards to ETH
079:         if (totalSupply == 0)
080:             preDepositPrice = 10 ** 18; // initializes with a price of 1
081:         else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
082:
083:         uint256 totalStakeValueEth = 0; // total amount of derivatives worth of ETH in system
084:         for (uint i = 0; i < derivativeCount; i++) {
085:             uint256 weight = weights[i];
086:             IDerivative derivative = derivatives[i];
087:             if (weight == 0) continue;
088:             uint256 ethAmount = (msg.value * weight) / totalWeight;
089:
090:             // This is slightly less than ethAmount because slippage
091:             uint256 depositAmount = derivative.deposit{value: ethAmount}();
092:             uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
093:                 depositAmount
094:             ) * depositAmount) / 10 ** 18;
095:             totalStakeValueEth += derivativeReceivedEthValue;
096:         }
097:         // mintAmount represents a percentage of the total assets in the system
098:         uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
099:         _mint(msg.sender, mintAmount);
100:         emit Staked(msg.sender, msg.value, mintAmount);
101:     }

```

## **G-04 Cheaper to use uint256 as boolean values**

When you want to store a true/false value, the first instinct is naturally to store it as a boolean. However, because the EVM stores boolean values as a uint8 type, which takes up two bytes, it is actually more expensive to access the value. This is because it EVM words are 32 bytes long, so extra logic is needed to tell the VM to parse a value that is smaller than standard.The most efficient way to store true/false values, then, is to use 1 for false and 2 for true.

https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L63-L232-L244

```solidity
File: contracts\SafeEth_orig.sol
232:     function setPauseStaking(bool _pause) external onlyOwner {
233:         pauseStaking = _pause;
234:         emit StakingPaused(pauseStaking);
235:     }
---
241:     function setPauseUnstaking(bool _pause) external onlyOwner {
242:         pauseUnstaking = _pause;
243:         emit UnstakingPaused(pauseUnstaking);
244:     }

```

## Impact

Adding an address not implementing (or incorrectly implementing) the IDerivative interface (owner could by mistake copy-paste the wrong address, like an EOA or a totally different contract, or forget a function in its new Derivative contract) will cause a DoS of all (or some) operations.

As it can be seen in the code snippet, no verification is done on the added derivative :