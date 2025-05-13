## NOTE-01 - `generate.rs` still uses the deprecated `requestWithCallback` method

```rust
File: src/command/generate.rs
14: pub async fn generate(opts: &GenerateOptions) -> Result<()> {
15:     let contract = Arc::new(
16:         SignablePythContract::from_config(
17:             &Config::load(&opts.config.config)?.get_chain_config(&opts.chain_id)?,
18:             &opts.private_key,
19:         )
20:         .await?,
21:     );
22: 
23:     let user_randomness = rand::random::<[u8; 32]>();
24:     let provider = opts.provider;
25: 
26:     let mut last_block_number = contract.provider().get_block_number().await?;      
27:     tracing::info!(block_number = last_block_number.as_u64(), "block number");
28: 
29:     tracing::info!("Requesting random number...");
30: 
31:     // Request a random number on the contract
32:     let sequence_number = contract
33:         .request_with_callback_wrapper(&provider, &user_randomness)  <@ deprecated method
34:         .await?;
```