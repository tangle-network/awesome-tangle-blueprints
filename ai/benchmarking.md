# Cursor Rules: Blueprint Benchmarking Guide

This document defines the structure of the benchmarking system implemented for Tangle Blueprints, which is used to determine pricing and collect metrics from running services.

---

## 1. Benchmarking Lifecycle
- **Four measurement points**:
  - Available Resources: Baseline system capabilities
  - Pre-execution: State before service launch
  - During Execution: Runtime consumption
  - Post-execution: State after completion

- **Two phases**:
  - Operator Registration: Initial hardware benchmarks
  - Service Execution: Continuous monitoring during runtime

---

## 2. Resource Metrics
| Resource | Metrics | Priority |
|----------|---------|----------|
| CPU | Usage %, Time, Cores, Load | High |
| Memory | RSS, Heap, Virtual, Peak | High |
| Disk I/O | IOPS, Throughput, Latency | Medium |
| Network | Bandwidth, Connections | Medium |
| GPU | Utilization, Memory | Low* |

Direct mapping to pricing configuration:
```
{ kind = "CPU", count = 1, price_per_unit_rate = 0.001 }
{ kind = "MemoryMB", count = 1024, price_per_unit_rate = 0.00005 }
{ kind = "StorageMB", count = 1024, price_per_unit_rate = 0.00002 }
```

---

## 3. Pricing Engine Integration
1. Initial hardware benchmarking during operator registration
2. Results cached by blueprint ID
3. Quote generation workflow:
   - Retrieve benchmark profile
   - Apply resource requirements
   - Calculate: `Resource Cost × TTL Adjustment × Security Adjustment`
   - Create security commitments
   - Sign quote's hash with operator's key

---

## 4. Pricing Calculation
```
Operator Price = Resource Cost × TTL Adjustment × Security Adjustment
```

- Resource Cost: Based on benchmarked capabilities and usage
- TTL Adjustment: Service duration factor
- Security Adjustment: Based on asset exposure commitments

---

## 5. Benchmark Caching
- Stores results by blueprint ID
- Updates when requirements change
- Fast retrieval for quote generation
- Prevents redundant benchmark runs
- Ensures pricing consistency

---

## 6. Monitoring Tools
Will integrate with one of the following to collect metrics of running services:
1. `sysinfo`: Cross-platform metrics
2. `heim`: Async-first monitoring
3. `procfs`: Linux-specific, detailed metrics
4. `tokio-metrics`: Runtime task metrics
5. `prometheus`: Metrics exposure

---

**Note:** Benchmarking integrates with pricing engine to provide accurate cost estimates based on actual resource capabilities.
