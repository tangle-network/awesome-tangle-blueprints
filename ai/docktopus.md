# Cursor Rules: Docktopus Crate (Docker Container Management)

This document provides clear patterns and conventions for using the `docktopus` Rust crate for creating, managing, and orchestrating Docker containers within Blueprints.

Follow these rules to ensure idiomatic Rust usage, clear integration patterns, and robust container lifecycle management.

---

## 1. Container Initialization Pattern

Docker containers are initialized via the `Container` struct using a fluent builder pattern. This ensures a consistent and readable configuration process.

**Example Initialization:**
```rust
let container = Container::new(docker_client, "image/name:tag")
    .env(["KEY=value"])
    .cmd(["/bin/command", "arg1", "arg2"])
    .binds(["./host/path:/container/path:ro"])
    .port_bindings(port_map)
    .runtime("runtime")
    .restart_policy(RestartPolicy {
        name: Some(RestartPolicyNameEnum::UNLESS_STOPPED),
        ..Default::default()
    });
```

Always chain methods to configure your container clearly before calling `.create()` or `.start()`.

---

## 2. Container Lifecycle Management

Containers should follow a predictable lifecycle:

- **Initialization → Create → Start → (optional monitoring or waiting) → Stop → Remove**

### ✅ Correct Lifecycle Example:
```rust
let mut container = Container::new(client, "my/image")
    .cmd(["run-service"])
    .create()
    .await?;

container.start(false).await?;
container.wait().await?;
container.stop().await?;
container.remove(None).await?;
```

### ❌ Common Mistakes:
- Do **not** skip `.create()` if you plan to manually control startup.
- Do **not** call `.remove()` without first stopping or providing `force`.

---

## 3. Context Integration Pattern

Containers used within Blueprints should typically be managed within a dedicated context struct. Include the Docker client (`Arc<Docker>`) alongside your `Container` instance.

### Recommended Context Layout:
```rust
#[derive(Clone)]
pub struct DockerContext {
    pub docker: Arc<Docker>,
    pub container: Container,
}
```

### Context Initialization Example:
```rust
impl DockerContext {
    pub async fn new(docker: Arc<Docker>, image: &str) -> Result<Self> {
        let container = Container::new(docker.clone(), image)
            .cmd(["serve"])
            .create()
            .await?;

        Ok(Self { docker, container })
    }
}
```

---

## 4. Container Status Monitoring

Regularly check container status via the provided `.status()` method, allowing your Blueprint to react to container state changes.

### Status Check Example:
```rust
match container.status().await? {
    Some(status) if status.is_active() => {
        // Container is running normally.
    },
    Some(status) if !status.is_usable() => {
        // Container is in a problematic state; remove it forcefully.
        container.remove(Some(RemoveContainerOptions { force: true, ..Default::default() })).await?;
    },
    _ => {
        // Container status unknown or container not created yet.
    },
}
```

---

## 5. Volume Bindings

Volume bindings mount directories or files from the host system into the Docker container, allowing for shared data or persistent storage.

Bindings use the standard Docker format: `"host_path:container_path[:options]"`.

- `host_path`: Path on your local host.
- `container_path`: Path within the container where the host volume is mounted.
- `options` (optional): e.g., `ro` (read-only), `rw` (read-write).

### Example:
```rust
.binds([
    "./local/dir:/data",
    "/var/log:/logs:ro",
])
```

This example mounts:
- The local `./local/dir` to `/data` in the container (read-write by default).
- The host `/var/log` directory to `/logs` in the container as read-only (`ro`).

---

## 6. Port Mappings

Port mappings expose container ports to the host system, making internal services accessible externally.

They are defined using a `HashMap<String, Option<Vec<PortBinding>>>`:

- **Key format:** `"container_port/protocol"`, e.g., `"80/tcp"`.
- **Value:** Vector of `PortBinding` specifying host IP and port.

### Example:
```rust
let mut port_bindings = HashMap::new();
port_bindings.insert(
    "8080/tcp".into(),
    Some(vec![PortBinding {
        host_ip: Some("127.0.0.1".into()),
        host_port: Some("80".into()),
    }]),
);

let container = Container::new(client, "my/webserver")
    .port_bindings(port_bindings);
```

In this example:
- Container port `8080/tcp` is exposed on host IP `127.0.0.1`, port `80`.

---

## 7. Restart Policies

Docker restart policies define container behavior upon exit or failure. By default, Docker containers have **no restart policy** (they do not restart automatically). Specify explicitly if automatic restarts are desired:

Available restart policies:
- **`ALWAYS`**: Always restart the container.
- **`UNLESS_STOPPED`**: Restart unless explicitly stopped by the user.
- **`ON_FAILURE`**: Restart only if the container exits with an error.
- **No policy (default)**: Container will not restart automatically.

### Example (optional):
```rust
.restart_policy(RestartPolicy {
    name: Some(RestartPolicyNameEnum::ON_FAILURE),
    maximum_retry_count: Some(5),
})
```

---

## 8. Adding Custom `/etc/hosts` Entries (`extra_hosts`)

Docker containers can include custom host-to-IP mappings in their `/etc/hosts` file via the `.extra_hosts()` method.

This method accepts entries formatted as `"hostname:IP"` pairs. Useful for accessing host or custom DNS mappings from within containers.

### Example:
```rust
.extra_hosts([
    "my-host.local:192.168.1.5",
    "host.docker.internal:host-gateway",
])
```

- Adds two host mappings:
    - `my-host.local` → `192.168.1.5`
    - `host.docker.internal` → Docker's host gateway IP (allows the container to access the host system easily)

This feature is entirely optional and context-specific; include it only as needed.

---

## 9. Testing Container Operations

Write integration tests using your host’s local Docker server (via `bollard`) to verify the functionality of your container lifecycle operations.

### Correct Test Example:
```rust
#[tokio::test]
async fn container_lifecycle_test() -> Result<(), docktopus::container::Error> {
    // Connect to the local Docker daemon on the host machine.
    let docker = DockerBuilder::new().await?;
    let mut container = Container::new(docker.client(), "alpine:latest")
        .cmd(["echo", "test"])
        .create()
        .await?;

    container.start(true).await?;
    assert_eq!(container.status().await?, Some(ContainerStatus::Exited));

    container.remove(None).await?;
    Ok(())
}
```

This test:
- Creates and runs an Alpine container that echoes `"test"`.
- Waits for completion, verifies exit status, and cleans up afterward.

---

## 10. Recommended Blueprint Context Integration

Integrate Docker containers directly into your Blueprint's context struct, allowing your jobs to manage containers seamlessly.

### Example Context Struct:
```rust
#[derive(Clone)]
pub struct MyBlueprintContext {
    #[config]
    pub env: BlueprintEnvironment,
    pub docker: Arc<Docker>,
    pub container: Container,
}
```

### Context Initialization:
```rust
impl MyBlueprintContext {
    pub async fn new(env: BlueprintEnvironment, image: &str) -> Result<Self> {
        let docker = DockerBuilder::new().await?;
        let container = Container::new(docker.client(), image).create().await?;

        Ok(Self {
            env,
            docker: docker.client(),
            container,
        })
    }
}
```

This setup allows your Blueprint handlers and jobs to manage Docker containers via the Blueprint's standard context injection mechanisms.

---

## 11. Don'ts ❌

- **Do NOT instantiate a new Docker client for each container.**  
  Always share via `Arc<Docker>`.

- **Do NOT silently ignore errors.**  
  Propagate errors clearly.

- **Do NOT rely on implicit Docker behavior.**  
  Specify all critical container options explicitly.

---

By following these Rules, your integration of Docker container management within blueprints will remain idiomatic, maintainable, and robust.