# 🔴 H01 - Attacker can copy valid message and call `solana-vault::oapp_lz_receive` with arbitrary ctx.accounts allowing him to steal tokens from `vault_deposit_wallet`

### Summary
An attacker can to listen for message sent on source chain, and execute them with crafted inputs on the destination chain to drain the protocol owned token account.

The `solana-vault::oapp_lz_receive` instruction can be called by anyone, as no verification is done on the transaction signer. Also, no checks are performed regarding the content of the message, and the actual token accounts provided with the instructions and involved in the token transfers.

The transfer is done from the `vault_deposit_wallet` (owned by PDA owned by the solana-vault program) to the `user_deposit_wallet` (which isn't verified), this allows an attacker to unauthorizedly drain USDC tokens from the `vault_deposit_wallet`

This allows an attacker to listen for message sent from the source chain, and front-run on the destination chain the Executor call to `solana-vault::oapp_lz_receive`, by replicating the content of the sent message, but maliciously swapped the `user_deposit_wallet` account provided in the instruction context, which don't match the expected receiver of the message.

_Also, it seems that in the [current implementation](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract/blob/main/solana-vault/packages/solana/contracts/programs/uln/src/instructions/commit_verification.rs#L14-L14) of the Uln program, the attacker can proceed with the verification himself as there is no restriction on the caller, but here we don't even need that hypothesis as our attacker can simply wait for the verification to be commited to then call the `solana-vault::oapp_lz_receive` function._

See how, in contrast, [another project](https://github.com/mahmudsudo/zealot_Nft/blob/e447eedcfa8254dc2abe940422249375fb6b0bd5/programs/zealot_nft/src/instructions/lz_receive.rs#L43) checks the receiving wallet against the message param

### Root Cause
- `solana-vault::oapp_lz_receive` can be called by anyone
- no check ensuring that the transfer correspond to the received message
- transfer is executed [using a PDA signer](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract/blob/main/solana-vault/packages/solana/contracts/programs/solana-vault/src/instructions/oapp_instr/oapp_lz_receive.rs#L121) owned by the calling contract, meaning anyone can execute the transfer if no specific verification checks are done.

### Internal pre-conditions
- Victim request a withdrawal from source chain
- Message get sucessfully verified on destination chain

### External pre-conditions
- Attacker detect victim withdrawal request and extract the message
- Attacker calls `solana-vault::oapp_lz_receive` before the Executor with malicious inputs

### Attack path
1. Alice request a withdrawal
2. The message is constructed on the EVM Oapp (source chain) and sent through LZ to the Solana Oapp (destination chain)
	- at this point, the attacker sees the transaction on the source chain and extract the message
3. The message get [verified](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract/blob/main/solana-vault/packages/solana/contracts/programs/uln/src/instructions/commit_verification.rs#L30)
	- this [update the `payload_hash.hash`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/7bcfb4d5dac4192570af5e51dbc67413a6116a14/packages/layerzero-v2/solana/programs/programs/endpoint/src/instructions/verify.rs#L79) which is necessary when clear is called [(1)](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract/blob/main/solana-vault/packages/solana/contracts/programs/solana-vault/src/instructions/oapp_instr/oapp_lz_receive.rs#L79-L79)[(2)](https://github.com/LayerZero-Labs/LayerZero-v2/blob/7bcfb4d5dac4192570af5e51dbc67413a6116a14/packages/layerzero-v2/solana/programs/programs/endpoint/src/instructions/oapp/clear.rs#L53-L56) during the `solana-vault::oapp_lz_receive` instruction  
4. Once the message gets verified the attacker call `solana-vault::oapp_lz_receive` with the extracted message but provide its own [`user_deposit_wallet`](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract/blob/main/solana-vault/packages/solana/contracts/programs/solana-vault/src/instructions/oapp_instr/oapp_lz_receive.rs#L40) account rather than Alice's one
5. the transfer is executed and the attacker receive the funds 

### Impact
Loss of funds
This also break the [following README property](https://github.com/sherlock-audit/2024-09-orderly-network-solana-contract?tab=readme-ov-file#q-what-propertiesinvariants-do-you-want-to-hold-even-if-breaking-them-has-a-lowunknown-impact)

### PoC
-

### Mitigation
Ensure that `user_deposit_wallet` is the one expected in the message payload.