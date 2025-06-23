# Cursor Rules: Quality of Service (QoS) Integration Guide

This document explains how to integrate and use the Blueprint SDK's Quality of Service (QoS) system to add comprehensive observability, monitoring, and dashboard capabilities to any Blueprint. QoS provides unified metrics collection, log aggregation, heartbeat monitoring, and visualization through a cohesive interface.

---

## 1. QoS Overview

The Blueprint QoS system provides a complete observability stack:
- **Heartbeat Service**: Sends periodic heartbeats to Tangle to prevent slashing
- **Metrics Collection**: Captures system and application metrics
- **Logging**: Aggregates logs via Loki for centralized querying
- **Dashboards**: Creates Grafana visualizations automatically
- **Server Management**: Optionally runs containerized instances of Prometheus, Loki, and Grafana

The QoS system is designed to be added to any Blueprint type (Tangle, Eigenlayer, P2P, or Cron) as a background service.

---

## 2. Integrating QoS into a Blueprint

### Main Blueprint Setup
```rust
#[tokio::main]
async fn main() -> Result<(), blueprint_sdk::Error> {
    let env = BlueprintEnvironment::load()?;
    
    // Create your Blueprint's primary context
    let context = MyContext::new(env.clone()).await?;
    
    // Configure QoS system
    let qos_config = blueprint_qos::default_qos_config();
    let heartbeat_consumer = Arc::new(MyHeartbeatConsumer::new());
    
    // Standard Blueprint runner setup with QoS
    BlueprintRunner::builder(TangleConfig::default(), env)
        .router(Router::new()
            .route(JOB_ID, handler.layer(TangleLayer))
            .with_context(context))
        .producer(producer)
        .consumer(consumer)
        .qos_service(qos_config, Some(heartbeat_consumer))
        .run()
        .await
}
```

### Implementing HeartbeatConsumer
```rust
#[derive(Clone)]
struct MyHeartbeatConsumer {
    // Add any required fields for heartbeat submission
}

impl HeartbeatConsumer for MyHeartbeatConsumer {
    fn consume_heartbeat(
        &self,
        service_id: u64,
        blueprint_id: u64,
        metrics_data: String,
    ) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
        // Implement custom heartbeat logic here, specific to blueprint
        Ok(())
    }
}
```

---

## 3. QoS Configuration

### Using Default Configuration
The simplest way to get started is with the default configuration:

```rust
let qos_config = blueprint_qos::default_qos_config();
```

This initializes a configuration with:
- Heartbeat service (disabled until configured)
- Metrics collection
- Loki logging
- Grafana integration
- Automatic server management set to `false`

### Custom Configuration

Customize the configuration for your specific needs:

```rust
let qos_config = QoSConfig {
    heartbeat: Some(HeartbeatConfig {
        service_id: Some(42),
        blueprint_id: Some(7),
        interval_seconds: 60,
        jitter_seconds: 5,
    }),
    metrics: Some(MetricsConfig::default()),
    loki: Some(LokiConfig::default()),
    grafana: Some(GrafanaConfig {
        endpoint: "http://localhost:3000".into(),
        admin_user: Some("admin".into()),
        admin_password: Some("admin".into()),
        folder: None,
    }),
    grafana_server: Some(GrafanaServerConfig::default()),
    loki_server: Some(LokiServerConfig::default()),
    prometheus_server: Some(PrometheusServerConfig::default()),
    docker_network: Some("blueprint-network".into()),
    manage_servers: true,
    service_id: Some(42),
    blueprint_id: Some(7),
    docker_bind_ip: Some("0.0.0.0".into()),
};
```

### Using the Builder Pattern
The builder pattern provides a fluent API for configuration:

```rust
let qos_service = QoSServiceBuilder::new()
    .with_heartbeat_config(HeartbeatConfig {
        service_id: Some(service_id),
        blueprint_id: Some(blueprint_id),
        interval_seconds: 60,
        jitter_seconds: 5,
    })
    .with_heartbeat_consumer(Arc::new(consumer))
    .with_metrics_config(MetricsConfig::default())
    .with_loki_config(LokiConfig::default())
    .with_grafana_config(GrafanaConfig::default())
    .with_prometheus_server_config(PrometheusServerConfig {
        host: "0.0.0.0".into(),
        port: 9090,
        ..Default::default()
    })
    .manage_servers(true)
    .with_ws_rpc_endpoint(ws_endpoint)
    .with_keystore_uri(keystore_uri)
    .build()?;
```

---

## 4. Recording Blueprint Metrics and Events

### Job Performance Tracking

Track job execution and performance in your job handlers:

```rust
pub async fn process_job(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<u64>> {
    let start_time = std::time::Instant::now();
    
    // Process the job
    let result = perform_processing(&data)?;
    
    // Record job execution metrics
    if let Some(qos) = &ctx.qos_service {
        qos.record_job_execution(
            JOB_ID,
            start_time.elapsed().as_secs_f64(),
            ctx.service_id,
            ctx.blueprint_id
        );
    }
    
    Ok(TangleResult::Success(result))
}
```

### Error Tracking

Track job errors for monitoring and alerts:

```rust
match perform_complex_operation() {
    Ok(value) => Ok(TangleResult::Success(value)),
    Err(e) => {
        if let Some(qos) = &ctx.qos_service {
            qos.record_job_error(JOB_ID, "complex_operation_failure");
        }
        Err(e.into())
    }
}
```

---

## 5. Automatic Dashboard Creation

QoS can automatically create Grafana dashboards that display your Blueprint's metrics:

```rust
// Create a custom dashboard for your Blueprint
if let Some(mut qos) = qos_service {
    if let Err(e) = qos.create_dashboard("My Blueprint") {
        error!("Failed to create dashboard: {}", e);
    } else {
        info!("Created Grafana dashboard for My Blueprint");
    }
}
```

The dashboard includes:
- System resource usage (CPU, memory, disk, network)
- Job execution metrics (frequency, duration, error rates)
- Log visualization panels (when Loki is configured)
- Service status and uptime information

---

## 6. Accessing QoS in Context

Typically, you'll want to store the QoS service in your Blueprint context:

```rust
#[derive(Clone)]
pub struct MyContext {
    #[config]
    pub env: BlueprintEnvironment,
    pub data_dir: PathBuf,
    pub qos_service: Option<Arc<QoSService<MyHeartbeatConsumer>>>,
    pub service_id: u64,
    pub blueprint_id: u64,
}

impl MyContext {
    pub async fn new(env: BlueprintEnvironment) -> Result<Self, Error> {
        // Initialize QoS service
        let qos_service = initialize_qos(&env)?;
        
        Ok(Self {
            data_dir: env.data_dir.clone().unwrap_or_else(default_data_dir),
            qos_service: Some(Arc::new(qos_service)),
            service_id: 42,
            blueprint_id: 7,
            env,
        })
    }
}
```

You can then access the QoS service in your job handlers:

```rust
pub async fn my_job(
    Context(ctx): Context<MyContext>,
    TangleArg(data): TangleArg<String>,
) -> Result<TangleResult<()>> {
    // Access QoS metrics provider
    if let Some(qos) = &ctx.qos_service {
        if let Some(provider) = qos.provider() {
            let cpu_usage = provider.get_cpu_usage()?;
            info!("Current CPU usage: {}%", cpu_usage);
        }
    }
    
    // Job implementation
    Ok(TangleResult::Success(()))
}
```

---

## 7. Server Management

QoS can automatically manage Grafana, Prometheus, and Loki servers:

```rust
// Configure server management
let qos_config = QoSConfig {
    grafana_server: Some(GrafanaServerConfig {
        port: 3000,
        container_name: "blueprint-grafana".into(),
        image: "grafana/grafana:latest".into(),
        ..Default::default()
    }),
    loki_server: Some(LokiServerConfig {
        port: 3100,
        container_name: "blueprint-loki".into(),
        image: "grafana/loki:latest".into(),
        ..Default::default()
    }),
    prometheus_server: Some(PrometheusServerConfig {
        port: 9090,
        container_name: "blueprint-prometheus".into(),
        image: "prom/prometheus:latest".into(),
        host: "0.0.0.0".into(),
        ..Default::default()
    }),
    docker_network: Some("blueprint-network".into()),
    manage_servers: true,
    ..Default::default()
};
```

For proper operation with Docker containers, ensure:
1. Your application binds metrics endpoints to `0.0.0.0` (not `127.0.0.1`)
2. Prometheus configuration uses `host.docker.internal` to access host metrics
3. Docker is installed and the user has the necessary permissions
4. A common Docker network is used for all containers

---

## 8. Do's and Don'ts

✅ DO:
- Initialize QoS early in your Blueprint's startup sequence
- Add QoS as a background service using `BlueprintRunner::background_service()`
- Record job execution metrics for all important jobs
- Use `#[derive(Clone)]` for your `HeartbeatConsumer` implementation
- Access QoS APIs through your Blueprint's context

❌ DON'T:
- Don't create separate QoS instances for different components
- Don't use hardcoded admin credentials in production code
- Don't pass the QoS service directly between jobs; use the context pattern
- Don't forget to bind Prometheus metrics server to `0.0.0.0` for Docker accessibility
- Don't ignore QoS shutdown or creation errors; they may indicate more serious issues

---

## 9. Quick Reference: QoS Components

| Component | Primary Struct | Config | Purpose |
|-----------|---------------|--------|---------|
| Unified Service | `QoSService` | `QoSConfig` | Main entry point for QoS integration |
| Heartbeat | `HeartbeatService` | `HeartbeatConfig` | Sends periodic liveness signals to chain |
| Metrics | `MetricsService` | `MetricsConfig` | Collects system and application metrics |
| Logging | N/A | `LokiConfig` | Configures log aggregation to Loki |
| Dashboards | `GrafanaClient` | `GrafanaConfig` | Creates and manages Grafana dashboards |
| Server Management | `ServerManager` | Various server configs | Manages Docker containers for observability stack |
