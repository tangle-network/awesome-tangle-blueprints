# Cursor Rules: Pricing Engine Architecture

This guide defines the core components and patterns of the Tangle Network Pricing Engine. Follow these guidelines to ensure your implementation is consistent, maintainable, and follows established patterns.

---

## 1. Pricing Engine Workflow

The Pricing Engine follows this operational flow:

1. **Operator Registration**: Operators register with a blueprint on-chain, providing their RPC server address (the pricing engine's address).

2. **Quote Request**: When a user wants to run a blueprint instance, they request quotes from all registered operators.

3. **Quote Calculation**: Each operator's pricing engine server:
   - Receives the request via gRPC
   - Verifies the proof-of-work to prevent DoS attacks
   - Retrieves the blueprint's benchmark profile from cache
   - Calculates pricing based on resource requirements and TTL
   - Creates security commitments based on requirements
   - Signs the quote with the operator's private key
   - Returns the signed quote to the user

4. **Operator Selection**: The user selects operators based on quotes, typically choosing the cheapest options. Future implementations may include tiers for better resources at higher prices.

5. **Service Request**: The user submits an on-chain request with the selected quotes, which serves as automatic approval from the operators.

6. **Blueprint Execution**: Operators run the blueprint as per the standard execution flow.

---

## 2. Price Calculation Model

The pricing model follows a multi-factor formula:

**For each operator:**
```
Operator Price = Resource Cost × TTL Adjustment × Security Adjustment
```

**Total Cost:**
```
Total Cost = Σ Operator Price
```

Where, for each operator:
- **Resource Cost** = resource_count × price_per_unit_rate
- **TTL Adjustment** = time_blocks × BLOCK_TIME
- **Security Adjustment** = Factor based on security requirements

---

## 3. Security Requirements & Commitments

Security requirements specify exposure percentages for assets:

- **Minimum Exposure**: Lowest acceptable percentage (0-100)
- **Maximum Exposure**: Highest acceptable percentage (0-100)
- **Asset Types**: Custom (u64) or ERC20 (H160 address)

When an operator receives a quote request with security requirements, they create a commitment that specifies the exact exposure percentage they guarantee to maintain (typically the minimum exposure percentage from the requirements).

---

## 4. Quote Signing & Verification

The pricing engine uses cryptographic signatures to ensure quote authenticity:

1. **Quote Creation**: The engine generates QuoteDetails with pricing information
2. **Hashing**: It creates a deterministic hash using protobuf serialization and SHA-256
3. **Signing**: The hash is signed with the operator's private key
4. **Verification**: Users verify signatures using the operator's public key

This ensures quotes cannot be tampered with and are binding for both operators and users.

---

## 5. Proof-of-Work System

The proof-of-work system prevents DoS attacks on pricing engines:

1. **Challenge Generation**: Based on blueprint ID and timestamp
2. **Proof Generation**: Computationally expensive operation performed by clients
3. **Verification**: Quick verification performed by the pricing engine
4. **Response Proof**: The engine includes its own proof in responses

---

## 6. Benchmark Caching

The pricing engine maintains a persistent cache of benchmark profiles for blueprints:

1. **Profile Storage**: Benchmark results are stored by blueprint ID
2. **Profile Retrieval**: Cached profiles are used for pricing calculations
3. **Profile Updates**: Profiles are updated when blueprint requirements change

This avoids redundant benchmark runs and ensures consistent pricing.

---

## 7. Pricing Configuration Format

Pricing configurations use TOML with blueprint-specific and default pricing:

- Default pricing applies to all blueprints without specific configurations
- Blueprint-specific pricing overrides default values for particular blueprints
- Each resource type (CPU, Memory, etc.) has its own pricing parameters

The pricing engine loads configuration from a TOML file with this structure:

```toml
# Default pricing for all blueprints
[default]
resources = [
  { kind = "CPU", count = 1, price_per_unit_rate = 0.001 },
  { kind = "MemoryMB", count = 1024, price_per_unit_rate = 0.00005 },
  { kind = "StorageMB", count = 1024, price_per_unit_rate = 0.00002 },
  { kind = "NetworkEgressMB", count = 1024, price_per_unit_rate = 0.00003 },
  { kind = "NetworkIngressMB", count = 1024, price_per_unit_rate = 0.00001 },
  { kind = "GPU", count = 1, price_per_unit_rate = 0.005 }
]

# Blueprint-specific pricing (overrides default)
[123]  # Blueprint ID
resources = [
  { kind = "CPU", count = 1, price_per_unit_rate = 0.0012 },
  { kind = "MemoryMB", count = 2048, price_per_unit_rate = 0.00006 }
]
```

---

## 8. gRPC API

The pricing engine exposes a gRPC API with these key components:

- **GetPrice**: Main endpoint for price quotes
- **Request**: Contains blueprint ID, TTL, proof-of-work, and security requirements
- **Response**: Contains signed quote details with total cost and resource breakdown

All messages use Protocol Buffers defined in a proto file. The complete type definitions include:

```protobuf
// The pricing service definition
service PricingEngine {
  // Retrieves a signed price quote for a given blueprint
  rpc GetPrice (GetPriceRequest) returns (GetPriceResponse);
}

// Asset type enumeration
enum AssetType {
  CUSTOM = 0;
  ERC20 = 1;
}

// Asset type definition
message Asset {
  oneof asset_type {
    // Custom asset with a numeric identifier
    uint64 custom = 1;
    // ERC20 token with an H160 address
    bytes erc20 = 2;
  }
}

// Security requirements for an asset
message AssetSecurityRequirements {
  // The asset type
  Asset asset = 1;
  // Minimum exposure percentage (0-100)
  uint32 minimum_exposure_percent = 2;
  // Maximum exposure percentage (0-100)
  uint32 maximum_exposure_percent = 3;
}

// Security commitment for an asset
message AssetSecurityCommitment {
  // The asset type
  Asset asset = 1;
  // Committed exposure percentage (0-100)
  uint32 exposure_percent = 2;
}

// Resource requirement for a specific resource type
message ResourceRequirement {
  // Resource kind (CPU, Memory, GPU, etc.)
  string kind = 1;
  // Quantity required
  uint64 count = 2;
}

// Pricing for a specific resource type
message ResourcePricing {
  // Resource kind (CPU, Memory, GPU, etc.)
  string kind = 1;
  // Quantity of the resource
  uint64 count = 2;
  // Price per unit in USD with decimal precision
  double price_per_unit_rate = 3;
}

// Request message for GetPrice RPC
message GetPriceRequest {
  // The blueprint ID
  uint64 blueprint_id = 1;
  // Time-to-live for service in blocks
  uint64 ttl_blocks = 2;
  // Proof of work to prevent DDOS
  bytes proof_of_work = 3;
  // Optional resource recommendations
  repeated ResourceRequirement resource_requirements = 4;
  // Security requirements for assets
  AssetSecurityRequirements security_requirements = 5;
}

// Response message for GetPrice RPC
message GetPriceResponse {
  // The quote details
  QuoteDetails quote_details = 1;
  // Signature of the hash of the body
  bytes signature = 2;
  // Operator ID
  bytes operator_id = 3;
  // Proof of work response
  bytes proof_of_work = 4;
}

// The detailed quote information
message QuoteDetails {
  // The blueprint ID
  uint64 blueprint_id = 1;
  // Time-to-live for service in blocks
  uint64 ttl_blocks = 2;
  // Total cost in USD with decimal precision
  double total_cost_rate = 3;
  // Timestamp when quote was generated
  uint64 timestamp = 4;
  // Expiry timestamp
  uint64 expiry = 5;
  // Resource pricing details
  repeated ResourcePricing resources = 6;
  // Security commitments for assets
  AssetSecurityCommitment security_commitments = 7;
}
```

---

## 9. Integration with Blockchain Events

The pricing engine listens for relevant blockchain events:

1. **Blueprint Registration**: Updates pricing when new blueprints are registered
2. **Price Target Updates**: Adjusts pricing when target prices change
3. **Service Requests**: Processes service requests with valid quotes

This ensures pricing remains synchronized with on-chain state.

---

## 10. Do's and Don'ts

**DO**:
- Always generate a valid proof-of-work for each quote request
- Verify operator signatures before accepting quotes
- Check quote expiry times before submitting on-chain
- Request quotes from all registered operators for best pricing
- Include complete security requirements in your requests

**DON'T**:
- Don't submit expired quotes to the chain
- Don't skip signature verification for any quotes
- Don't modify quote details after receiving them
- Don't reuse proof-of-work values across different requests
- Don't assume all operators will provide the same price
