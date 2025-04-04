# Cursor Rules: Shared Concepts for All Blueprints

This guide defines the foundational patterns shared across all Blueprint modalities (Tangle, Eigenlayer, Cron, P2P). Follow these to ensure your implementation is idiomatic, composable, and testable.

---

## 1. Blueprint Runner Pattern
All Blueprints are launched via `BlueprintRunner::builder(...)`. This runner:
- Initializes the runtime.
- Starts a producer stream.
- Listens for jobs via the `Router`.
- Optionally handles graceful shutdown or background tasks.

```rust
BlueprintRunner::builder(config, env)
    .router(Router::new()
        .route(JOB_ID, handler.layer(...))
        .with_context(ctx))
    .producer(...)
    .consumer(...) // Tangle or EVM
    .background_service(...) // optional
    .with_shutdown_handler(...) // optional
    .run()
    .await?;
```

The config passed (e.g. `TangleConfig`, `EigenlayerBLSConfig`) determines how jobs are submitted to the chain—not where events are ingested from.

---

## 2. Router and Job Routing
Routers map Job IDs to handler functions. Each `.route(ID, handler)` must be unique.

Use `.layer(...)` to apply:
- `TangleLayer` (standard substrate filters)
- `FilterLayer::new(MatchesServiceId(...))` for multi-tenant service execution
- `FilterLayer::new(MatchesContract(...))` to scope EVM jobs by contract address

Use `.with_context(...)` to pass your context into jobs.

```rust
Router::new()
    .route(SOME_JOB_ID, do_something.layer(TangleLayer))
    .always(process_packet.layer(FilterLayer::new(MatchesContract(address!()))))
    .with_context(MyContext { ... })
```

---

## 3. Context Pattern
All contexts must:
- Wrap `BlueprintEnvironment` with `#[config]`
- Derive traits like `TangleClientContext`, `ServicesContext`, `KeystoreContext` as needed
- Optionally contain internal clients (Docker, RPC, gRPC, etc.)

Example:
```rust
#[derive(Clone, TangleClientContext, ServicesContext)]
pub struct MyContext {
    #[config]
    pub env: BlueprintEnvironment,
    pub data_dir: PathBuf,
    pub connection: Arc<DockerBuilder>,
    pub signer: TanglePairSigner,
}
```

Construction should be async:
```rust
impl MyContext {
    pub async fn new(env: BlueprintEnvironment) -> Result<Self> { ... }
}
```

---

## 4. Producer + Consumer Compatibility
Your producer and consumer determine event ingestion and message submission:

| Producer Type     | Source                     | Usage Modality     |
|------------------|----------------------------|--------------------|
| `TangleProducer` | Finalized Substrate blocks | Tangle-only        |
| `PollingProducer`| EVM `eth_getLogs` polling  | EVM/Tangle Hybrid  |
| `CronJob`        | Internal time-based tick   | All modal options  |
| `RoundBasedAdapter` | P2P message queue     | P2P/Networking/MPC  |

| Consumer Type     | Role                           | Notes                  |
|------------------|--------------------------------|-------------------------|
| `TangleConsumer` | Submits signed jobs to Tangle  | Only for Tangle chains |
| `EVMConsumer`    | Sends txs via Alloy wallet     | Valid in Tangle configs |

🧠 **Important:** A Blueprint using `TangleConfig` may use EVM producers + consumers. The config determines *where results are sent*, not *where events come from*.

---

## 5. Job Signature Conventions
Use extractors to simplify job argument handling:

- `TangleArg<T>`: one field
- `TangleArgs2<A, B>`: two fields
- `BlockEvents`: EVM logs
- `Context<MyContext>`: context injection

Return `TangleResult<T>` or `Result<(), Error>` depending on job type.

```rust
pub async fn handler(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<u64>> {
    ...
}
```

---

## 6. Keystore and Signer Usage
Load from `BlueprintEnvironment`:
```rust
let key = env.keystore().first_local::<SpEcdsa>()?;
let secret = env.keystore().get_secret::<SpEcdsa>(&key)?;
let signer = TanglePairSigner::new(secret.0);
```

For BLS (Eigenlayer):
```rust
let pubkey = ctx.keystore().first_local::<ArkBlsBn254>()?;
let secret = ctx.keystore().expose_bls_bn254_secret(&pubkey)?.unwrap();
let bls = BlsKeyPair::new(secret.to_string())?;
```

---

## 7. Naming & Organization
- Job IDs are declared as `pub const JOB_NAME_ID: u64 = 0;`
- Handlers should be snake_case with suffixes (`_eigen`, `_local`, `_cron`, etc.)
- Contexts use `PascalCaseContext` naming (e.g., `AggregatorContext`)
- Group jobs into modules/files like `jobs/mod.rs`, `jobs/indexer.rs`, `jobs/config.rs`

Use `#[debug_job]` macro to log entry and exit automatically.

---

## 8. Testing Conventions
Use `TangleTestHarness` or `Anvil` + Alloy to simulate:
- Service creation (`setup_services::<N>()`)
- Job submission (`submit_job(...)`)
- Execution polling (`wait_for_job_execution(...)`)
- Result validation (`verify_job(...)`)

For Eigenlayer:
- Use `cast` CLI or Anvil state
- Watch logs via Alloy `watch_logs`
- Load contracts with `sol!` macro bindings

---

## 9. Don'ts
❌ Never use a `TangleConsumer`, `TangleProducer` outside of a Tangle specific blueprint.

