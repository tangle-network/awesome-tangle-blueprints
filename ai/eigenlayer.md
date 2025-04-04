# Cursor Rules: Eigenlayer Blueprint Guide

## 1. What is an Eigenlayer Blueprint?
Eigenlayer Blueprints are services built on top of EVM-compatible chains using Alloy for contract interaction and log parsing. They run offchain logic in response to onchain events and submit responses back via APIs, BLS aggregators, or onchain transactions.

Key components:
- **PollingProducer**: Streams logs using `eth_getLogs` from an EVM endpoint.
- **Alloy**: Re-exported via `blueprint_sdk::alloy` for decoding events, sending transactions, and managing EVM keys.
- **Job Router**: Maps job IDs to handlers reacting to EVM events.
- **Contexts**: Usually contain RPC clients, wallets, aggregator clients, BLS keys, and task managers.

---

## 2. Project Skeleton

```rust
#[tokio::main]
async fn main() -> Result<(), blueprint_sdk::Error> {
    let env = BlueprintEnvironment::load()?;

    let signer = AGGREGATOR_PRIVATE_KEY.parse()?;
    let wallet = EthereumWallet::from(signer);
    let provider = get_wallet_provider_http(&env.http_rpc_endpoint, wallet.clone());

    let context = CombinedContext::new(
        EigenSquareContext { client: ..., std_config: env.clone() },
        Some(AggregatorContext::new(...).await?),
        env.clone(),
    );

    let task_producer = PollingProducer::new(
        Arc::new(provider),
        PollingConfig::default().poll_interval(Duration::from_secs(1)),
    ).await?;

    BlueprintRunner::builder(EigenlayerBLSConfig::new(...), env)
        .router(Router::new()
            .route(XSQUARE_JOB_ID, xsquare_eigen)
            .route(INITIALIZE_TASK_JOB_ID, initialize_bls_task)
            .with_context(context))
        .producer(task_producer)
        .background_service(aggregator_context)
        .run()
        .await
}
```

---

## 3. Jobs and Event Decoding

Handlers receive EVM logs as `BlockEvents`, use Alloy to decode them:

```rust
pub async fn xsquare_eigen(
    Context(ctx): Context<CombinedContext>,
    BlockEvents(events): BlockEvents,
) -> Result<(), TaskError> {
    let task_events = events.iter().filter_map(|log| {
        NewTaskCreated::decode_log(&log.inner, true).ok().map(|e| e.data)
    });

    for task in task_events { ... }
    Ok(())
}
```

### Event Source Type:
```rust
sol!(
    #[derive(Debug)]
    NewTaskCreated,
    "contracts/out/TaskManager.sol/TaskManager.json"
);
```
Use `#[sol(rpc)]` for binding contract ABI to log/event decoding or contract calls.

---

## 4. Contexts in Eigenlayer
Common context components:
- Aggregator gRPC client
- Keystore (for BLS key extraction)
- Wallet (EVM private key)
- Task manager address

```rust
#[derive(Clone)]
pub struct CombinedContext {
    pub eigen_context: EigenSquareContext,
    pub aggregator_context: Option<AggregatorContext>,
    pub env: BlueprintEnvironment,
}
```

You must wrap clients and keys in the context to make them accessible in jobs.

---

## 5. BLS Signing
Use `ctx.keystore().first_local::<ArkBlsBn254>()?` to load your operator key.
Then use `eigensdk::crypto_bls::BlsKeyPair` to produce BLS signatures.

```rust
let key = BlsKeyPair::new(secret.to_string())?;
let sig = key.sign_message(keccak256(...));
```
Use `operator_id_from_g1_pub_key(...)` to generate your operator ID from the public key.

---

## 6. Job Naming & Convention
- Job IDs are `u32` constants: `pub const XSQUARE_JOB_ID: u32 = 0;`
- Use `#[debug_job]` for logging execution context.
- Name jobs based on the domain: `initialize_bls_task`, `xsquare_eigen`, etc.

---

## 7. Testing: Anvil + Harness
Enable the `testing` feature in SDK to use in-process `anvil` EVM testnets:

```rust
let (testnet1, testnet2) = spinup_anvil_testnets().await?;

let tempdir = setup_temp_dir(...)?;
let harness = TangleTestHarness::setup(tempdir).await?;
```

Run full end-to-end jobs via the Blueprint API and dispatch messages using Alloy:

```rust
let tx = mailbox.dispatch_2(31338, recipient, Bytes::from("Hello"))
    .send().await?;
```

Then validate EVM events using `watch_logs()`:

```rust
let stream = provider.watch_logs(&filter).await?.into_stream();
```

---

## 8. Do’s and Don’ts
✅ DO:
- Use `PollingProducer` for all EVM-based jobs.
- Use Alloy’s `sol!` macro for ABI + log decoding.
- Derive all routing context traits.
- Use BLS keys and aggregator gRPC clients from SDK.

❌ DON'T:
- Never use `TangleProducer` or `TangleConsumer` in Eigenlayer blueprints.
- Avoid manually decoding raw event data—use Alloy decode_log helpers.
- Avoid re-implementing BLS or crypto logic—use the SDK-provided abstractions.

---
