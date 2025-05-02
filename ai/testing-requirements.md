# AI Prompt: Blueprint Testing Requirements

This document outlines the testing requirements and best practices for all Tangle Blueprint projects.

---

## 1. Test Location & Coverage

- Tests **MUST** reside in `blueprint/tests/` or within job modules at `blueprint/src/jobs/tests.rs`
- Every job handler (`fn` routed in `main.rs`) **MUST** have at least one corresponding integration test
- Tests should cover both success paths and potential error conditions

## 2. Testing Frameworks

| Blueprint Type | Framework to Use |
|----------------|-----------------|
| Tangle | `TangleTestHarness` |
| EVM/Hybrid | `Anvil` + `alloy` or `TangleTestHarness` (depending on primary interaction) |
| Docker | Test with local Docker daemon via `bollard`/`docktopus` |

---

## 3. TangleTestHarness Pattern

All Tangle Blueprint tests should follow this standard pattern:

### Setup
```rust
#[tokio::test]
async fn test_my_job() -> Result<(), Box<dyn std::error::Error>> {
    // Create a temporary directory for test data
    let temp_dir = tempfile::tempdir()?;
    
    // Initialize the test harness
    let harness = TangleTestHarness::setup(temp_dir.path()).await?;
    
    // Set up N services (1 in this example)
    let (mut test_env, service_id, _) = harness.setup_services::<1>(false).await?;
    
    // Initialize the test environment
    test_env.initialize().await?;
    
    // Register the job handler with appropriate layer
    test_env.add_job(my_job_handler.layer(TangleLayer)).await;
    
    // Start the test environment
    test_env.start(()).await?;
    
    // ... test execution and verification steps ...
    
    Ok(())
}
```

### Job Execution
```rust
// Submit a job with appropriate input values
let call = harness.submit_job(
    service_id, 
    MY_JOB_ID, 
    vec![InputValue::Uint64(5)]
).await?;

// Wait for the job to complete execution
let result = harness.wait_for_job_execution(service_id, call).await?;
```

### Verification
```rust
// Verify the job result matches expected output
harness.verify_job(&result, vec![OutputValue::Uint64(25)]);

// Assert success
assert!(result.is_success());

// Verify side effects if applicable
assert!(temp_dir.path().join("output.json").exists());
```

---

## 4. Docker Container Testing

When testing jobs that interact with Docker containers:

```rust
#[tokio::test]
async fn container_lifecycle_test() -> Result<(), docktopus::container::Error> {
    // Connect to local Docker daemon
    let docker = DockerBuilder::new().await?;
    
    // Create a test container
    let mut container = Container::new(docker.client(), "alpine:latest")
        .cmd(["echo", "test"])
        .create()
        .await?;
    
    // Start and wait for completion
    container.start(true).await?;
    
    // Verify container status
    assert_eq!(container.status().await?, Some(ContainerStatus::Exited));
    
    // Clean up
    container.remove(None).await?;
    
    Ok(())
}
```

### Key Requirements
- Tests must verify the complete container lifecycle: creation, start, status checks, stop, removal
- Ensure proper cleanup even if tests fail (use `defer` patterns or `Drop` implementations)
- Test both success paths and error handling for Docker operations

---

## 5. Mocking External Dependencies

For tests that depend on external services:

- Use in-memory implementations where possible
- Consider Docker containers for database dependencies
- Mock HTTP APIs and other external services
- Ensure tests are deterministic and don't depend on external state

---

## 6. Enforcement Rules

- **MUST** include integration tests for all job handlers
- **MUST** use `TangleTestHarness` for Tangle Blueprints
- **MUST** follow the setup → register → execute → verify pattern
- **MUST** clean up resources (like Docker containers) created during tests
- **MUST** verify both job results (`OutputValue`) and any significant side effects
- **MUST** ensure tests can run in CI environments
- **SHOULD** run tests in isolation (not depending on other test outcomes)
- **SHOULD** include both positive tests (valid inputs) and negative tests (error conditions)
