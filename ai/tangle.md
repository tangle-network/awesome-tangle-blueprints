# Cursor Rules: Tangle Blueprint Guide

## 1. What is a Tangle Blueprint?
A Tangle Blueprint is a modular, job-executing service built on top of Substrate (Tangle) using the Blueprint SDK. It is structured similarly to a microservice with:

- **Job Router**: Maps numeric job IDs to logic handlers
- **BlueprintRunner**: Core executor that ties together producer, consumer, router, and context
- **TangleProducer**: Streams finalized blocks/events from a Tangle RPC endpoint
- **TangleConsumer**: Signs and sends results back to the chain
- **Context**: Manages local state (data directory, docker containers, keystore)

These services are composable and deterministic, often containerized (e.g., Docker) and can be tested using the built-in `TangleTestHarness`.

## 2. Project Skeleton
The canonical `main.rs` structure looks like:

```rust
#[tokio::main]
async fn main() -> Result<(), sdk::Error> {
    let env = BlueprintEnvironment::load()?;

    let signer = env.keystore().first_local::<SpSr25519>()?;
    let pair = env.keystore().get_secret::<SpSr25519>(&signer)?;
    let signer = TanglePairSigner::new(pair.0);

    let client = env.tangle_client().await?;
    let producer = TangleProducer::finalized_blocks(client.rpc_client.clone()).await?;
    let consumer = TangleConsumer::new(client.rpc_client.clone(), signer);

    let context = MyContext::new(env.clone()).await?;

    BlueprintRunner::builder(TangleConfig::default(), env)
        .router(Router::new()
            .route(JOB_ID, handler.layer(TangleLayer))
            .with_context(context))
        .producer(producer)
        .consumer(consumer)
        .run()
        .await
}
```

## 3. Job Composition
### Handler Signature
Handlers take a context and deserialized args:

```rust
pub async fn set_config(
    Context(ctx): Context<MyContext>,
    TangleArgs2(Optional(config_urls), origin_chain_name): TangleArgs2<
        Optional<List<String>>,
        String,
    >,
) -> Result<TangleResult<u64>> {
    // Implementation
}
```

Use `TangleArg`, `TangleArgs2`, etc. for parsing input fields. Always return `TangleResult<T>`.

### Event Filters
Apply `TangleLayer` or `MatchesServiceId` to jobs to filter execution by service identity.

## 4. Context Composition

The context needs to include the BlueprintEnvironment as seen below. Any other components should be added as needed per blueprint.

```rust
#[derive(Clone, TangleClientContext, ServicesContext, KeystoreContext)]
pub struct MyContext {
    #[config]
    pub env: BlueprintEnvironment,
    pub data_dir: PathBuf,
}

impl MyContext {
    pub async fn new(env: BlueprintEnvironment) -> Result<Self> {
        Ok(Self {
            data_dir: env.data_dir.clone().unwrap_or_else(default_data_dir),
            env,
        })
    }
}
```

Contexts should:
- Derive required traits for routing
- Contain DockerBuilder or other service-level state if needed
- Wrap fs, keystore, or networking state

## 5. Job Naming & IDs
- Job IDs: `pub const MY_JOB_ID: u64 = 0;`
- Handler naming: `snake_case_action_target` (e.g., `spawn_indexer_local`)
- Files: Group jobs in a `jobs` module, one file per logical task
- Use `#[debug_job]` macro for helpful traces

## 6. Testing Blueprints
Use `TangleTestHarness` to simulate a full node and runtime:

```rust
let harness = TangleTestHarness::setup(temp_dir).await?;
let (mut test_env, service_id, _) = harness.setup_services::<1>(false).await?;
test_env.initialize().await?;
test_env.add_job(square.layer(TangleLayer)).await;
test_env.start(()).await?;

let call = harness.submit_job(service_id, 0, vec![InputValue::Uint64(5)]).await?;
let result = harness.wait_for_job_execution(service_id, call).await?;

harness.verify_job(&result, vec![OutputValue::Uint64(25)]);
```

Testing is composable, isolated, and persistent with `tempfile::TempDir`.

## 7. Key Implementation Rules
- MUST use `TangleProducer` and `TangleConsumer` for Tangle interaction
- MUST use `TanglePairSigner` initialized from the environment keystore
- MUST apply `TangleLayer` or other appropriate filters to job handlers
- MUST use `TangleArg`/`TangleArgsN` extractors for job inputs
- MUST return `TangleResult<T>` for jobs that report results back to the chain
- DO NOT manually decode block data; rely on extractors
- MUST handle errors gracefully using `Result`
- DO NOT use `unwrap()` or `expect()` in production code
- MUST store persistent data under `data_dir` from env
- MUST derive `KeystoreContext` and other required context traits
