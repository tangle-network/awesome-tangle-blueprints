# Cursor Rules: Blueprint SDK Import Paths

This document provides a comprehensive reference for importing components from the Blueprint SDK, organized by feature categories. Each section lists the available import paths and the feature flag required to enable them.

## Core Components (Always Available)

These components are always available regardless of which protocol you're targeting:

```rust
// Runner and Router - Core components for creating Blueprints
use blueprint_sdk::runner::BlueprintRunner;
use blueprint_sdk::runner::config::BlueprintEnvironment;
use blueprint_sdk::Router;

// Extractors - For handling job arguments
use blueprint_sdk::extract::Context;

// Error handling
use blueprint_sdk::Error;
use blueprint_sdk::error::RunnerError;

// Core context traits
use blueprint_sdk::contexts::keystore::KeystoreContext;

// Core types for job definitions
use blueprint_sdk::Job;
use blueprint_sdk::BackgroundService;
```

## Keystore Access (Always Available)

Keystore components for managing cryptographic keys:

```rust
// Keystore access
use blueprint_sdk::keystore::KeystoreBackend;
use blueprint_sdk::keystore::backends::Backend;
use blueprint_sdk::keystore::backends::bn254::Bn254Backend;

// Crypto primitives
use blueprint_sdk::crypto::sp_core::SpSr25519;
use blueprint_sdk::crypto::sp_core::SpEcdsa;
```

## Tangle Protocol Support (Feature: "tangle")

Components specific to Tangle blockchain integration:

```rust
// Main Tangle components
use blueprint_sdk::tangle::consumer::TangleConsumer;
use blueprint_sdk::tangle::producer::TangleProducer;
use blueprint_sdk::tangle::layers::TangleLayer;
use blueprint_sdk::tangle::filters::MatchesServiceId;
use blueprint_sdk::tangle::config::TangleConfig;

// Tangle Context
use blueprint_sdk::contexts::tangle::TangleClientContext;

// Tangle Signer
use blueprint_sdk::crypto::tangle_pair_signer::TanglePairSigner;

// Tangle Job Input Extraction
use blueprint_sdk::tangle::extract::TangleArg;
use blueprint_sdk::tangle::extract::TangleArgs2;
use blueprint_sdk::tangle::extract::TangleArgs3;
use blueprint_sdk::tangle::extract::Optional;
use blueprint_sdk::tangle::extract::List;

// Job results
use blueprint_sdk::tangle::result::TangleResult;

// Additional configuration
use blueprint_sdk::runner::tangle::config::TangleConfig;
```

## EVM Protocol Support (Feature: "evm")

Components for interacting with EVM-compatible blockchains:

```rust
// Alloy types (imported via blueprint_sdk)
use blueprint_sdk::alloy::network::EthereumWallet;
use blueprint_sdk::alloy::primitives::{Address, U256, keccak256};
use blueprint_sdk::alloy::sol_types::{SolEvent, SolType, SolValue};
use blueprint_sdk::alloy::signer_local::PrivateKeySigner;

// EVM producer components
use blueprint_sdk::evm::producer::{PollingConfig, PollingProducer};
use blueprint_sdk::evm::util::get_wallet_provider_http;

// EVM Extractors
use blueprint_sdk::evm::extract::BlockEvents;

// EVM Context
use blueprint_sdk::contexts::evm::EvmContext;
```

## Eigenlayer Support (Feature: "eigenlayer")

Components specifically for Eigenlayer AVS integration:

```rust
// Eigenlayer Configuration
use blueprint_sdk::runner::eigenlayer::bls::EigenlayerBLSConfig;

// Eigenlayer Crypto
use blueprint_sdk::crypto::bn254::ArkBlsBn254;

// EigenSDK Components
use blueprint_sdk::eigensdk::crypto_bls::BlsKeyPair;
use blueprint_sdk::eigensdk::types::operator::operator_id_from_g1_pub_key;

// Eigenlayer Context
use blueprint_sdk::contexts::eigenlayer::EigenlayerContext;
```

## Networking Support (Feature: "networking")

Components for peer-to-peer communication:

```rust
// Networking Core
use blueprint_sdk::networking::service::NetworkService;
use blueprint_sdk::networking::service_handle::NetworkServiceHandle;
use blueprint_sdk::networking::config::NetworkConfig;
use blueprint_sdk::networking::allowed_keys::AllowedKeys;
use blueprint_sdk::networking::allowed_keys::InstanceMsgPublicKey;
use blueprint_sdk::networking::message::MessageRouting;
use blueprint_sdk::networking::participant::ParticipantInfo;

// Context trait
use blueprint_sdk::contexts::networking::NetworkingContext;
```

## Round-based Protocol (Feature: "round-based-compat")

Components for round-based protocols (requires "networking" feature):

```rust
// Round-based components
use blueprint_sdk::networking::round_based_compat::adapter::RoundBasedAdapter;
use blueprint_sdk::networking::round_based_compat::producer::RoundBasedProducer;
```

## Cron Jobs (Feature: "cronjob")

Components for scheduled tasks:

```rust
// Cron job producer
use blueprint_sdk::producers::CronJob;
```

## Local Storage (Feature: "local-store")

Components for local key-value storage:

```rust
// Storage components
use blueprint_sdk::stores::Store;
use blueprint_sdk::stores::local::LocalStore;
```

## Macro Support (Feature: "macros")

Macros for attribute-based code generation:

```rust
// Context derive macros
use blueprint_sdk::macros::context::KeystoreContext;
use blueprint_sdk::macros::context::TangleClientContext;
use blueprint_sdk::macros::context::ServicesContext;
use blueprint_sdk::macros::context::NetworkingContext;

// Debug macros
use blueprint_sdk::macros::debug_job;

// FromRef derive macro
use blueprint_sdk::extract::FromRef;
```

## Logging (Feature: "tracing")

Components for logging:

```rust
// Logging macros
use blueprint_sdk::{debug, info, warn, error};
```

## Testing Utilities (Feature: "testing")

Components for testing Blueprints:

```rust
// Test harness
use blueprint_sdk::testing::chain_setup::TangleTestHarness;
use blueprint_sdk::testing::chain_setup::spinup_anvil_testnets;

// Temporary files
use blueprint_sdk::testing::tempfile::TempDir;
```

## Common Import Patterns

### Tangle Blueprint Main Imports

```rust
use blueprint_sdk::Router;
use blueprint_sdk::contexts::tangle::TangleClientContext;
use blueprint_sdk::crypto::sp_core::SpSr25519;
use blueprint_sdk::crypto::tangle_pair_signer::TanglePairSigner;
use blueprint_sdk::extract::Context;
use blueprint_sdk::keystore::backends::Backend;
use blueprint_sdk::runner::BlueprintRunner;
use blueprint_sdk::runner::config::BlueprintEnvironment;
use blueprint_sdk::runner::tangle::config::TangleConfig;
use blueprint_sdk::tangle::consumer::TangleConsumer;
use blueprint_sdk::tangle::filters::MatchesServiceId;
use blueprint_sdk::tangle::layers::TangleLayer;
use blueprint_sdk::tangle::producer::TangleProducer;
```

### Eigenlayer Blueprint Main Imports

```rust
use std::sync::Arc;
use std::time::Duration;

use blueprint_sdk::alloy::network::EthereumWallet;
use blueprint_sdk::alloy::primitives::Address;
use blueprint_sdk::alloy::signer_local::PrivateKeySigner;
use blueprint_sdk::evm::producer::{PollingConfig, PollingProducer};
use blueprint_sdk::evm::util::get_wallet_provider_http;
use blueprint_sdk::runner::BlueprintRunner;
use blueprint_sdk::runner::config::BlueprintEnvironment;
use blueprint_sdk::runner::eigenlayer::bls::EigenlayerBLSConfig;
use blueprint_sdk::Router;
```

### Job Handler Imports

#### Tangle Job Handler

```rust
use blueprint_sdk::extract::Context;
use blueprint_sdk::tangle::extract::TangleArg;
use blueprint_sdk::tangle::result::TangleResult;
use blueprint_sdk::macros::debug_job;
```

#### Eigenlayer Job Handler

```rust
use blueprint_sdk::alloy::primitives::{U256, keccak256};
use blueprint_sdk::alloy::sol_types::{SolEvent, SolType, SolValue};
use blueprint_sdk::contexts::keystore::KeystoreContext;
use blueprint_sdk::crypto::bn254::ArkBlsBn254;
use blueprint_sdk::evm::extract::BlockEvents;
use blueprint_sdk::extract::Context;
use blueprint_sdk::keystore::backends::Backend;
use blueprint_sdk::keystore::backends::bn254::Bn254Backend;
use blueprint_sdk::macros::debug_job;
use blueprint_sdk::eigensdk::crypto_bls::BlsKeyPair;
use blueprint_sdk::eigensdk::types::operator::operator_id_from_g1_pub_key;
```

## Feature Flag Requirements

This table shows which features you need to enable in your Cargo.toml for specific functionality:

| Functionality | Required Feature Flag |
|---------------|----------------------|
| Tangle support | "tangle" |
| EVM support | "evm" |
| Eigenlayer support | "eigenlayer" (also enables "evm") |
| P2P Networking | "networking" |
| Round-based protocols | "round-based-compat" (also needs "networking") |
| Cron jobs | "cronjob" |
| Local storage | "local-store" |
| Macros and derive | "macros" |
| Testing utilities | "testing" |

Example Cargo.toml configuration for a Tangle Blueprint:

```toml
[dependencies]
blueprint-sdk = { version = "0.1.0-alpha.7", features = ["tangle", "macros"] }
```

Example for an Eigenlayer Blueprint:

```toml
[dependencies]
blueprint-sdk = { version = "0.1.0-alpha.7", features = ["eigenlayer", "macros"] }
```
