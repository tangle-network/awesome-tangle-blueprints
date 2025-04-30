# Cursor Rules: Eigenlayer Blueprint Guide

## 1. What is an Eigenlayer Blueprint?
Eigenlayer Blueprints are services built on top of EVM-compatible chains using Alloy for contract interaction and log parsing. They run offchain logic in response to onchain events and submit responses via APIs, BLS aggregators, or onchain transactions.

Key components:
- **PollingProducer**: Streams logs using `eth_getLogs` from an EVM endpoint
- **Alloy**: Re-exported via `blueprint_sdk::alloy` for decoding events, sending transactions, and managing EVM keys
- **Job Router**: Maps job IDs to handlers reacting to EVM events
- **BlueprintRunner**: Core executor that configures producer, consumer, router, and context
- **Context**: Contains RPC clients, wallets, aggregator clients, BLS keys, and task managers

## 2. Project Skeleton

```rust
#[tokio::main]
async fn main() -> Result<(), blueprint_sdk::Error> {
    let env = BlueprintEnvironment::load()?;

    // Set up the wallet and provider
    let signer = PRIVATE_KEY.parse()?;
    let wallet = EthereumWallet::from(signer);
    let provider = get_wallet_provider_http(&env.http_rpc_endpoint, wallet.clone());

    // Set up the context with necessary components
    let context = CombinedContext::new(
        ClientContext { client: ..., env: env.clone() },
        Some(AggregatorContext::new(...).await?),
        env.clone(),
    );

    // Create a producer to poll for events
    let task_producer = PollingProducer::new(
        Arc::new(provider),
        PollingConfig::default().poll_interval(Duration::from_secs(1)),
    ).await?;

    // Set up the blueprint runner
    BlueprintRunner::builder(EigenlayerBLSConfig::new(...), env)
        .router(Router::new()
            .route(TASK_JOB_ID, task_handler)
            .route(INITIALIZE_TASK_JOB_ID, initialize_bls_task)
            .with_context(context))
        .producer(task_producer)
        .background_service(aggregator_context)
        .run()
        .await
}
```

## 3. Jobs and Event Decoding

Handlers receive EVM logs as `BlockEvents` and use Alloy to decode them:

```rust
pub async fn task_handler(
    Context(ctx): Context<CombinedContext>,
    BlockEvents(events): BlockEvents,
) -> Result<(), TaskError> {
    let task_events = events.iter().filter_map(|log| {
        NewTaskCreated::decode_log(&log.inner, true).ok().map(|e| e.data)
    });

    for task in task_events {
        // Process each task
    }
    Ok(())
}
```

### Contract/Event Binding
Use the `sol!` macro for binding contract ABIs:

```rust
sol!(
    #[allow(missing_docs, clippy::too_many_arguments)]
    #[sol(rpc)]
    #[derive(Debug, Serialize, Deserialize)]
    TaskManager,
    "contracts/out/TaskManager.sol/TaskManager.json"
);
```

## 4. Contexts in Eigenlayer
Contexts should include components specific to Eigenlayer operations:

```rust
#[derive(Clone, KeystoreContext)]
pub struct CombinedContext {
    pub client_context: ClientContext,
    pub aggregator_context: Option<AggregatorContext>,
    pub env: BlueprintEnvironment,
}
```

Common context components:
- Inner Contexts, if you have multiple - we can only have one main context used with the Runner (e.g. client, aggregator)
- Keystore (for BLS key extraction)
- Wallet/Signer (EVM private key)
- Task manager address

## 5. BLS Signing
Eigenlayer Blueprints use BLS signatures for aggregating operator responses:

```rust
// Load operator key
let pubkey = ctx.keystore().first_local::<ArkBlsBn254>()?;
let secret = ctx.keystore().expose_bls_bn254_secret(&pubkey)?.unwrap();

// Create a keypair and sign
let key = BlsKeyPair::new(secret.to_string())?;
let sig = key.sign_message(keccak256(...));
```

Use `operator_id_from_g1_pub_key(...)` to generate operator ID from public key.

## 6. Job Naming & Structure
- Job IDs are `u32` constants: `pub const TASK_JOB_ID: u32 = 0;`
- Use `#[debug_job]` for logging execution context
- Name handlers based on the domain: `initialize_bls_task`, `process_task_event`
- Group related jobs in modules under `src/jobs/`

## 7. Testing: Anvil + Harness
Enable the `testing` feature in SDK to use in-process Anvil EVM testnets:

```rust
// Set up test environment
let (testnet, _) = spinup_anvil_testnets().await?;
let tempdir = setup_temp_dir(...)?;
let harness = TangleTestHarness::setup(tempdir).await?;

// Deploy contracts and send transactions
let tx = mailbox.dispatch_2(31338, recipient, Bytes::from("Hello"))
    .send().await?;

// Watch for events and validate
let filter = Filter::new().address(vec![contract_address]);
let stream = provider.watch_logs(&filter).await?.into_stream();
```

## 8. Key Implementation Rules
- MUST use `PollingProducer` for all EVM-based jobs
- MUST use Alloy's `sol!` macro for ABI + log decoding
- MUST derive `KeystoreContext` trait
- MUST use BLS keys for operator signatures
- MUST validate event signatures and message sources
- DO NOT use `TangleProducer` or `TangleConsumer` in Eigenlayer blueprints
- DO NOT manually decode event data—use Alloy decode_log helpers
- DO NOT re-implement BLS or crypto logic—use SDK abstractions
- MUST handle errors gracefully using proper error types
- MUST properly clean up resources created during operation
