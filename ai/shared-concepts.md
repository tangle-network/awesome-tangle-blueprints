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

## 3. Context Pattern & State Management

### Structure & Definition
The Context struct is the primary container for application state and shared clients:

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

**Required Fields:**
- `#[config] pub env: BlueprintEnvironment` - Provides access to configuration, keystore, data directories - this is always required
- Other fields should contain the shared clients and state needed by job handlers - they will differ per Blueprint

**Required Traits:**
- `#[derive(Clone)]` - Contexts must be cloneable for job handlers - this is always required
- Chain-specific traits as needed:
  - `TangleClientContext` - For Tangle chain interactions
  - `ServicesContext` - For service registry access
  - `KeystoreContext` - For direct keystore access

### Initialization & Usage
Initialization must be async and handle errors appropriately:

```rust
impl MyContext {
    pub async fn new(env: BlueprintEnvironment) -> Result<Self> {
        // Initialize data directory
        let data_dir = env.data_dir.clone().unwrap_or_else(default_data_dir);
        
        // Set up shared clients
        let docker = Arc::new(DockerBuilder::new().await?.client());
        
        // Set up signer if needed
        let key = env.keystore().first_local::<SpEcdsa>()?;
        let secret = env.keystore().get_secret::<SpEcdsa>(&key)?;
        let signer = TanglePairSigner::new(secret.0);
        
        Ok(Self { env, data_dir, docker, signer })
    }
}
```

Job handlers access the context via the extractor pattern:

```rust
pub async fn my_handler(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<u64>> {
    // Access environment config, keystore, clients, etc.
    let config = &ctx.env.config;
    let client = ctx.env.tangle_client().await?;
    
    // Use shared clients
    let container = ctx.docker.create_container("image:tag", ...).await?;
    
    // Return result
    Ok(TangleResult::new(42))
}
```

### State Management Best Practices
- Use Context for sharing clients and long-lived state across handlers
- For service-specific persistent state, prefer:
  - Files in service-specific directories: `ctx.data_dir.join("service_id")`
- Avoid storing highly dynamic, per-job state directly in Context
- Wrap shared clients in `Arc` for thread-safe access

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

**Important:** A Blueprint using `TangleConfig` may use EVM producers + consumers. The config determines *where results are sent*, not *where events come from*.

---

## 5. Job Signature Conventions

### Handler Pattern
Job handlers use extractors for context and argument handling:

```rust
#[debug_job]
pub async fn handler_name(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
    // Other extractors as needed
) -> Result<TangleResult<U>, Error> {
    // Implementation
}
```

**Key Components:**
- **Extractors:** Use these to handle inputs automatically:

  General extractors:
  - `Context<MyContext>`: Injects the context

  Tangle-specific extractors:
  - `TangleArg<T>`: Single field argument
  - `TangleArgs2<A, B>`, `TangleArgs3<A, B, C>`: Multiple fields up to 15 arguments at TangleArgs15<..>
  - `Optional<T>`: For optional arguments
  - `List<T>`: For array/list arguments
  
  EVM-specific extractors:
  - `BlockEvents`: For extracting EVM logs

- **Return Types:**
  - `Result<TangleResult<T>, Error>`: For jobs submitting results to Tangle
  - `Result<(), Error>`: For jobs with no chain result

- **Filtering:** Apply appropriate layers to filter job execution:
  - `TangleLayer`: Standard Tangle job call filtering
  - `FilterLayer::new(MatchesServiceId(...))`: For multi-tenant filtering
  - `FilterLayer::new(MatchesContract(...))`: For EVM contract filtering

### Job Organization
- Define Job IDs as constants: `pub const MY_JOB_ID: u64 = 0;`
- Use `snake_case` for handler names, with suffixes indicating purpose (e.g., `handle_request_tangle`)
- Group related jobs in modules under `jobs/` directory
- Use the `#[debug_job]` macro for automatic logging of entry/exit

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

## 7. Key Architectural Concepts

### Modularity
- Blueprint services follow a strict separation between binary and library crates
- Binary crate handles only initialization of the runner
- Library crate contains all business logic, jobs, context, and utilities
- Code is organized into logical modules (jobs, context, utils, etc.)

### Microservice Pattern
- Each Blueprint is a self-contained service triggered by on-chain events (jobs)
- Each job has a clear, focused responsibility
- Service state is properly isolated and managed via the Context

### Producer/Consumer Flow
```
Producer → Router → Job Handlers → Consumer
(Events)   (Routes)  (Logic)      (Results)
```

### State Management
- Context struct manages shared state and clients 
- Service-specific state is persisted in data_dir or database
- Dynamic, per-job state should not be stored in main Context

## 8. Common Pitfalls to Avoid

| Don't | Do Instead |
|---------|-------------|
| Put logic in `bin` crate | Keep all app logic in the `lib` crate |
| Ignore errors from SDK | Propagate errors using `?` and handle appropriately |
| Create new Docker clients per operation | Initialize once in `Context::new` and share via `Arc<Docker>` |
| Rely on Docker defaults | Explicitly configure restart policies, resource limits, etc. |
| Use blocking operations in async handlers | Use `tokio::spawn_blocking` or async APIs |
| Create naming collisions with Job IDs | Ensure Job ID constants are unique and descriptive |
| Use incorrect Producer/Consumer pairs | Match Producer/Consumer to event source and target chain |

## 9. Development Process

1. **Understand Requirements:**
   - Review available documentation for current goals and status

2. **Structure:**
   - Adhere to folder architecture guidelines for organizing files
   - Follow the bin/lib crate separation

3. **Implementation:**
   - Write code following coding standards and best practices
   - Implement required integrations using appropriate patterns
   - Configure proper state management and context

4. **Testing:**
   - Add integration tests for all jobs
   - Test all interactions thoroughly
   - Verify error handling and edge cases

5. **Documentation:**
   - Update `README.md` with clear usage instructions
   - Document significant architectural decisions

---

## 10. Rust Coding Standards

### Error Handling
Blueprint error types use `thiserror`- Errors in Blueprints must derive `thiserror::Error`.

- Propagate errors using `Result<T, E>` and `?` operator
- Avoid `unwrap()` or `expect()` in production code
- Define custom error types implementing `std::error::Error` and `From` traits:
```rust
#[derive(Debug, thiserror::Error)]
pub enum MyError {
    #[error("Failed to process: {0}")]
    ProcessingError(String),
    #[error("Database error: {0}")]
    DatabaseError(#[from] DatabaseError),
}
```

### Job Handler Best Practices
```rust
#[debug_job]  // Automatic entry/exit logging - only needed for debugging
pub async fn handler_name(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<U>, Error> {
    // Implementation
}
```

### Code Organization
- Group related code in logical modules (`jobs/`, `context/`, `utils/`, `docker/`)
- Structure job modules by functionality (`jobs/mod.rs`, `jobs/create_project.rs`)
- Use `pub mod module_name;` in `lib.rs` to expose functionality

### Strict Rules
- **MUST** use Blueprint SDK patterns for runner, router, jobs, and context
- **MUST NOT** use `TangleConsumer`/`TangleProducer` outside Tangle-specific blueprints
- **MUST NOT** use `TangleArg`/`TangleArgsN` extractors outside Tangle blueprints
- **MUST NOT** use `EVMConsumer`/`PollingProducer` outside EVM-specific blueprints
- **MUST** handle errors gracefully using `Result`

---

## 11. Blueprint Enforcement Rules

This section consolidates all enforcement rules across Blueprint development domains.

### Project Structure
- **MUST** follow the required directory and file structure
- **MUST** place `BlueprintRunner` setup in the binary crate's `main.rs`
- **MUST** keep all application logic in the library crate
- **MUST** create one module per job in the library crate's `src/jobs/`
- **MUST** define a `Context` struct in the library crate's `src/context.rs`
- **MUST** keep smart contract code isolated in `/contracts`
- **MUST NOT** place logic in the binary crate besides initialization

### Context and State Management
- **MUST** include `#[config] pub env: BlueprintEnvironment` in Context
- **MUST** derive `Clone` and necessary SDK context traits
- **MUST** initialize shared clients within the `new` function
- **MUST** access context in jobs via the `Context<MyContext>` extractor

### Job Handlers
- **MUST** handle errors gracefully using `Result`
- **MUST NOT** manually decode block data; rely on extractors

### Producer/Consumer
- **MUST** match Producer/Consumer to event source and target chain

### Docker/Docktopus Integration
- **MUST** use the fluent builder pattern for container creation
- **MUST** follow the correct container lifecycle
- **MUST** integrate container management within the Blueprint `Context`
- **MUST** share the `Arc<Docker>` client
- **MUST** explicitly define necessary configurations
- **MUST NOT** rely on implicit Docker defaults
- **MUST NOT** ignore errors from `docktopus` operations

### Error Handling
- **MUST** propagate errors using `Result<T, E>` and `?` operator
- **MUST NOT** use `unwrap()` or `expect()` in production code
- **SHOULD** define custom error types implementing appropriate traits
