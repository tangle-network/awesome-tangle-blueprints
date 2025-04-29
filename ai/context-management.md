# AI Prompt: Context Definition and State Management

This document provides detailed guidance on defining, initializing, and using the Blueprint Context struct for state management.

---

## 1. Context Structure and Definition

The Context struct is the primary container for application state and clients:

```rust
#[derive(Clone, TangleClientContext, ServicesContext)]
pub struct MyContext {
    #[config]
    pub env: BlueprintEnvironment,
    pub data_dir: PathBuf,
    pub docker: Arc<Docker>,
    pub db_connection: Arc<DatabaseClient>,
    pub signer: TanglePairSigner,
}
```

**Required Fields:**
- `#[config] pub env: BlueprintEnvironment` - Provides access to configuration, keystore, data directories
- Other state fields specific to your blueprint's needs

**Required Traits:**
- `#[derive(Clone)]` - Contexts must be cloneable for job handlers
- SDK traits based on usage:
  - `TangleClientContext` - For Tangle client features
  - `ServicesContext` - For service registry interactions
  - `KeystoreContext` - For direct keystore access

---

## 2. Context Initialization

Contexts must be initialized with an async constructor:

```rust
impl MyContext {
    pub async fn new(env: BlueprintEnvironment) -> Result<Self> {
        // Initialize data directory
        let data_dir = env.data_dir.clone().unwrap_or_else(default_data_dir);
        
        // Set up shared clients
        let docker_builder = DockerBuilder::new().await?;
        let docker = Arc::new(docker_builder.client());
        
        // Initialize database connection
        let db_connection = Arc::new(Database::connect(&config).await?);
        
        // Set up signer if needed
        let key = env.keystore().first_local::<SpEcdsa>()?;
        let secret = env.keystore().get_secret::<SpEcdsa>(&key)?;
        let signer = TanglePairSigner::new(secret.0);
        
        Ok(Self {
            env,
            data_dir,
            docker,
            db_connection,
            signer,
        })
    }
}
```

**Key Requirements:**
- Initialization **MUST** be `async`
- Properly handle errors and return `Result`
- Initialize all fields within the `new()` method
- Use default values or fallbacks where appropriate

---

## 3. Context Usage in Job Handlers

The Context is injected into job handlers using extractors:

```rust
pub async fn my_job_handler(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<u64>> {
    // Access environment config
    let config = &ctx.env.config;
    
    // Access keystore
    let key = ctx.env.keystore().first_local::<SpEcdsa>()?;
    
    // Access Tangle client
    let client = ctx.env.tangle_client().await?;
    
    // Access Docker client
    let container = ctx.docker.create_container("image:tag", ...).await?;
    
    // Access other state
    let result = ctx.db_connection.query("SELECT * FROM data").await?;
    
    // Return result
    Ok(TangleResult::new(42))
}
```

---

## 4. State Management Best Practices

- Use the Context as the primary mechanism for sharing state across handlers
- Wrap shared clients in `Arc` for thread-safe concurrent access
- For service-specific persistent state, consider:
  - Files in service-specific directories: `ctx.data_dir.join("service_id")`
  - Database storage keyed by service ID
  - State files with JSON/TOML configurations
- Avoid storing highly dynamic, per-job state directly in the Context struct
- Use temporary directories for job-specific artifacts

---

## 5. Enforcement Rules

- **MUST** define context struct in `blueprint/src/context.rs`
- **MUST** include `#[config] pub env: BlueprintEnvironment`
- **MUST** derive `Clone` and necessary SDK context traits
- **MUST** have an `async fn new(...) -> Result<Self>` constructor
- **MUST** initialize shared clients within the `new` function
- **MUST** access context in handlers via the `Context<MyContext>` extractor
