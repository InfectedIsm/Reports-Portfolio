# 🔴 H01 - The proposal typehash differ from the encoded struct for `questionId`
https://github.com/sherlock-audit/2024-09-predict-fun/blob/main/predict-dot-loan/contracts/PredictDotLoan.sol#L809-L809

### Summary
In `hashProposal` typehash string we can see `uint256 questionId` while in the struct it is `bytes32 questionId`, making the hashProposal wrong in regard to EIP712
This has not been

### Root Cause
wrong type hashed for `questionId`

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack path
N/A

### Impact
- Non compliance with EIP712 which break README statement: "The contract is expected to strictly comply with EIP-712 and EIP-1271."
- Will cause issues when processing signatures that uses the correct typehash string

# PoC
N/A

### Mitigation
Update the typehash with the correct type `bytes32` for `questionId`