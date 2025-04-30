# Cursor Rules: CronJobs with Blueprint SDK

This document describes how to use `CronJob` from the Blueprint SDK to run scheduled jobs within any Tangle or Eigenlayer Blueprint.

CronJobs generate `JobCall`s at a fixed time interval using cron expressions **that include seconds**. **Note that this differs from standard cron expressions.**

> **Important**: When using `CronJob`, ensure that the `cronjob` feature is explicitly enabled for the `blueprint-sdk` dependency in your project's `Cargo.toml`:
> ```toml
> blueprint-sdk = { version = "x.y.z", features = ["cronjob"] }
> ```

---

## 1. What is a CronJob?

A `CronJob` is a producer that implements `Stream<Item = JobCall>`. It triggers job execution at a configured interval.

**Cron-compatible jobs must not carry any parameters other than an optional `Context<T>`; external parameters outside the context are not permitted.**

```rust
// Creates a CronJob that triggers MY_JOB_ID every 5 seconds
let mut cron = CronJob::new(MY_JOB_ID, "*/5 * * * * *").await?;
```

---

## 2. Cron Schedule Format

Format: `sec min hour day month day-of-week`
- Example: `"*/10 * * * * *"` → every 10 seconds

**Note**: Blueprint SDK cron expressions **require seconds to be included**. Standard 5-field cron formats (without seconds) will not work correctly.

---

## 3. Using CronJob in a BlueprintRunner

```rust
// Set up a BlueprintRunner with a router and a CronJob producer that triggers every 30 seconds
BlueprintRunner::builder(config, env)
    .router(
        Router::new()
            .route(MY_JOB_ID, my_job) // Registers job `my_job` for job ID MY_JOB_ID
            .with_context(ctx) // Provides the optional context to the job
    )
    .producer(
        CronJob::new(MY_JOB_ID, "*/30 * * * * *").await? // CronJob configured to trigger every 30 seconds
    )
    .run()
    .await;
```

---

## 4. Writing a Cron-Compatible Job

**The job must never require input data beyond an optional context:**

```rust
// Cron-compatible job handler that uses only a context and no other parameters
pub async fn my_handler(Context(ctx): Context<MyContext>) -> Result<TangleResult<()>> {
    sdk::info!("Cron triggered job!");
    Ok(TangleResult(()))
}
```

Note that a `Context` is not required for Cron-compatible jobs. Only add it if it is necessary for the Blueprint's function.

---

## 5. Timezone Customization

Use `CronJob::new_tz` to explicitly set a timezone:

```rust
use chrono::Utc;

// Creates a CronJob that triggers at the start of every hour (minute and second zero) explicitly in UTC
CronJob::new_tz(MY_JOB_ID, "0 0 * * * *", Utc).await?;
```

---

## 6. Testing Cron Schedules

Example test:
```rust
// Creates a CronJob scheduled every 2 seconds for testing purposes
let mut cron = CronJob::new(0, "*/2 * * * * *").await?;

// Records the current instant
let before = Instant::now();

// Awaits the next cron-triggered job call
let call = cron.next().await.unwrap()?;

// Records the instant after the job is triggered
let after = Instant::now();

// Checks that at least 2 seconds passed before the job call was triggered
assert!(after.duration_since(before) >= Duration::from_secs(2));
```

---

## 7. Use Cases

- Scheduled indexing (e.g., periodic data pull)
- Service health checks
- Retry logic on fixed intervals
- Epoch-based snapshots

---

## 8. Best Practices

Use cron when:
- Jobs need to run at regular intervals
- Input arguments are static or unnecessary
- Jobs only depend on context and never external or dynamic parameters

**Don’t use cron when:**
- Input parameters are dynamic or user-driven
- External parameters outside context are required
- You expect reactive behavior (use Tangle or EVM events instead)

---

CronJobs are lightweight, native-scheduled job producers ideal for heartbeat-like tasks or polling systems.