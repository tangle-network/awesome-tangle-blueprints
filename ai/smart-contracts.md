# Cursor Rules: Smart Contracts in Blueprints

## 1. Smart Contract Structure
Blueprints that interact with blockchains typically require smart contracts. These contracts follow a standard structure:

- **Tangle Blueprints**: Contracts inherit from `BlueprintServiceManagerBase`
- **Eigenlayer Blueprints**: Contracts implement AVS-specific interfaces
- **All Contracts**: Must compile with EVM version 'shanghai' for consistency

## 2. Project Structure
Smart contracts in a Blueprint should be organized as follows:

```
contracts/
  ├── src/                # Contract source files
  ├── test/               # Contract test files
  ├── out/                # Compilation output
  ├── script/             # Deployment scripts
  ├── cache/              # Contract compilation cache
  └── broadcast/          # Transaction broadcasts
```

## 3. Configuration Files
Every Blueprint with contracts must include:

### foundry.toml

#### Tangle Blueprints
```toml
[profile.default]
src = "contracts/src"
test = "contracts/test"
out = "contracts/out"
script = "contracts/script"
cache_path = "contracts/cache"
broadcast = "contracts/broadcast"
libs = ["dependencies"]
auto_detect_remappings = true
evm_version = 'shanghai'

[soldeer]
recursive_deps = true
remappings_location = "txt"
remappings_version = false

[dependencies]
tnt-core = "0.3.0"
```

#### Eigenlayer AVS Blueprints
```toml
[profile.default]
evm_version = "shanghai"
solc_version = "0.8.27"
gas_limit = 5000000000
deny_warnings = false
via_ir = false
optimizer = true
optimizer_runs = 100

src = "contracts/src"
test = "contracts/test"
out = "contracts/out"
script = "contracts/script"
cache_path = "contracts/cache"
broadcast = "contracts/broadcast"
libs = ["dependencies"]
remappings = [
    "@eigenlayer/=dependencies/eigenlayer-middleware-0.5.4/lib/eigenlayer-contracts/src/",
    "@eigenlayer-middleware/=dependencies/eigenlayer-middleware-0.5.4/",
    "eigenlayer-contracts/=dependencies/eigenlayer-middleware-0.5.4/lib/eigenlayer-contracts/",
    "forge-std-1.9.6/=dependencies/forge-std-1.9.6/",
]

[soldeer]
recursive_deps = true
remappings_location = "txt"
remappings_version = false

[dependencies]
eigenlayer-middleware = { version = "0.5.4", git = "https://github.com/Layr-Labs/eigenlayer-middleware", rev = "4d63f27247587607beb67f96fdabec4b2c1321ef" }
forge-std = "1.9.6"
"@openzeppelin-contracts" = "4.7.0"
```

### remappings.txt
Used to specify import paths for contract dependencies:

#### Tangle Blueprints
```
tnt-core/=dependencies/tnt-core/src/
```

#### Eigenlayer AVS Blueprints
Eigenlayer projects typically use inline remappings in foundry.toml rather than a separate remappings.txt file, but can use either approach.

## 4. Integration with Rust
Rust integration with contracts depends on the blueprint type:

### Tangle
- Direct calls to contracts using TangleConsumer
- Receiving events via TangleProducer

### Eigenlayer
- Contract events are processed using Alloy:
  ```rust
  sol!(
      TaskManager,
      "contracts/out/TaskManager.sol/TaskManager.json"
  );
  
  // In handler
  let events = log_entries.iter().filter_map(|log| {
      TaskManager::NewTaskCreated::decode_log(&log.inner, true).ok().map(|e| e.data)
  });
  ```

## 5. Contract Build Process
The build process should be automated:

### build.rs
```rust
fn main() {
    // Trigger a recompile whenever contract sources change
    println!("cargo:rerun-if-changed=contracts/src");
    println!("cargo:rerun-if-changed=remappings.txt");
    println!("cargo:rerun-if-changed=foundry.toml");
}
```

### Blueprint.json
Must reference contracts if used:
```json
{
  "contracts": ["contracts/src/ExperimentalBlueprint.sol"]
}
```

## 6. Key Requirements

### All Blueprints
- MUST specify `evm_version = 'shanghai'` in foundry.toml
- MUST have appropriate remappings configured
- MUST include contract dependencies in foundry.toml
- MUST use build.rs to detect contract changes
- MUST structure contracts in the standard Foundry layout
- MUST properly handle contract versioning
- DO NOT hardcode addresses; use configuration files or environment
- DO NOT manually decode contract events; use Alloy decode_log

### Tangle Blueprint Specific
- MUST inherit from `BlueprintServiceManagerBase` for service contracts
- MUST include `tnt-core` dependency
- MUST use `TangleConsumer` for contract interactions

### Eigenlayer AVS Specific
- MUST include Eigenlayer-specific dependencies
- MUST include OpenZeppelin contracts for standard interfaces
- MUST use the correct BLS signing implementation
- MUST configure proper remappings for Eigenlayer contracts
