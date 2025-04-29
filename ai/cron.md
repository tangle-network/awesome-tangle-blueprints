# Cursor Rules: CronJobs with Blueprint SDK

This document describes how to use `CronJob` from the Blueprint SDK to run scheduled jobs within any Tangle or Eigenlayer Blueprint.

CronJobs generate `JobCall`s at a fixed time interval using standard cron expressions (with seconds included).

---

## 1. What is a CronJob?

A `CronJob` is a producer that implements `Stream<Item = JobCall>`. It triggers job execution at a configured interval.

It does not carry any arguments — jobs triggered this way can only accept `Context<T>`.

```rust
let mut cron = CronJob::new(MY_JOB_ID, "*/5 * * * * *").await?;
```

---

## 2. Cron Schedule Format

Format: `sec min hour day month day-of-week`
- Example: `"*/10 * * * * *"` → every 10 seconds

Note: Seconds **must be included** in Blueprint's cron parser.

---

## 3. Using CronJob in a BlueprintRunner

```rust
BlueprintRunner::builder(config, env)
    .router(Router::new().route(MY_JOB_ID, my_handler).with_context(ctx))
    .producer(CronJob::new(MY_JOB_ID, "*/30 * * * * *").await?)
    .run()
    .await;
```

---

## 4. Writing a Cron-Compatible Job

The job must not require any input data — only a context:

```rust
pub async fn my_handler(Context(ctx): Context<MyContext>) -> Result<TangleResult<()>> {
    sdk::info!("Cron triggered job!");
    Ok(TangleResult(()))
}
```

---

## 5. Timezone Customization

Use `CronJob::new_tz` to explicitly set a timezone:

```rust
use chrono::Utc;
CronJob::new_tz(MY_JOB_ID, "0 0 * * * *", Utc).await?;
```

---

## 6. Testing Cron Schedules

Example test:
```rust
let mut cron = CronJob::new(0, "*/2 * * * * *").await?;
let before = Instant::now();
let call = cron.next().await.unwrap()?;
let after = Instant::now();
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
- Input args are static or unnecessary

Don’t use cron when:
- Input parameters are dynamic or user-driven
- You expect reactive behavior (use Tangle or EVM events instead)

---

CronJobs are lightweight, native-scheduled job producers ideal for heartbeat-like tasks or polling systems.

