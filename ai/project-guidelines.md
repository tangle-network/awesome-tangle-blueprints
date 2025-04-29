# AI Prompt: General Project Guidelines & Goals

This document provides high-level context, goals, and project guidance for Blueprint development, helping maintain focus on critical aspects while avoiding common pitfalls.

---

## 1. Project Goals

The primary goal is to create a Tangle Blueprint service that manages an MCP (Multi-Party Computation) server running inside a Docker container. Key responsibilities include:

- Initializing and configuring Docker containers based on job inputs
- Managing the container lifecycle (create, start, stop, remove)
- Monitoring container health and status
- Implementing resource tiers (Small/Medium/Large) for container configuration
- Properly handling job inputs and returning outputs via `TangleResult`

### Core Jobs

- `create_project`: Starts the MCP server container, configuring it based on inputs
- `destroy_project`: Stops and removes the MCP server container and associated state
- Additional jobs may be added for monitoring, reconfiguration, etc.

---

## 2. Key Architectural Concepts

### Modularity
- Follow the strict separation between `bin` and `lib` crates
- Keep all business logic within the `blueprint` library crate
- Use modules to organize related functionality

### Microservice Pattern
- Treat the Blueprint as a self-contained service triggered by on-chain events (jobs)
- Each job should have a clear, focused responsibility
- Service state should be properly isolated and managed

### Producer/Consumer Flow
```
TangleProducer → Router → Job Handlers → TangleConsumer
  (Events)      (Routes)    (Docker)      (Results)
```

### State Management
- Use the `Context` struct for shared state and clients
- Persist service-specific state (container IDs, configuration) in:
  - Files within the `data_dir`
  - Database entries keyed by service ID
- Avoid storing highly dynamic, per-job state directly in the main `Context`

---

## 3. Common Pitfalls to Avoid

| ❌ Don't | ✅ Do Instead |
|---------|-------------|
| Put logic in `bin` crate | Keep all app logic in the `lib` crate |
| Ignore errors from SDK, docktopus, etc. | Propagate errors using `?` and handle appropriately |
| Create new Docker clients per operation | Initialize once in `Context::new` and share via `Arc<Docker>` |
| Rely on Docker defaults | Explicitly configure restart policies, resource limits, etc. |
| Manually parse block data | Use `TangleArg`/`TangleArgsN` extractors |
| Use blocking operations in async handlers | Use `tokio::spawn_blocking` or async APIs |
| Create naming collisions with Job IDs | Ensure Job ID constants are unique and descriptive |
| Use incorrect Producer/Consumer pairs | Match Producer/Consumer to event source and target chain |

---

## 4. Development Process

1. **Understand Requirements:**
   - Refer to `productContext.md` and `activeContext.md` for current goals and status

2. **Structure:**
   - Adhere to `projectStructure.md` for folder and file organization
   - Follow the bin/lib crate separation

3. **Implementation:**
   - Write code following coding standards and best practices
   - Implement Docker integration using `docktopus` patterns
   - Configure proper state management and context

4. **Testing:**
   - Add integration tests for all jobs
   - Test Docker interactions thoroughly
   - Verify error handling and edge cases

5. **Documentation:**
   - Update `README.md` with clear usage instructions
   - Document significant architectural decisions

---

## 5. Focus Areas for Implementation

- **Docker Management:**
  - Robust container lifecycle handling
  - Proper error handling for all container operations
  - Status monitoring and health checks

- **Resource Tiers:**
  - Implement Small/Medium/Large configurations
  - Set appropriate CPU, memory, and storage limits
  - Allow tier selection via job inputs

- **Monitoring:**
  - Implement health checks for container status
  - Consider SSE endpoints for live monitoring (if applicable)

- **Job Handling:**
  - Correctly extract and validate job inputs
  - Return appropriate `TangleResult` responses
  - Maintain proper error propagation

- **Testing:**
  - Write comprehensive integration tests
  - Cover both success and failure scenarios
  - Test Docker interactions with mocks or test containers
