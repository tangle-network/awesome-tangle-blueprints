# Build Script Guidelines for Tangle Blueprints

Every Tangle Blueprint must include a `build.rs` script in the binary crate root (`{{name}}-bin/build.rs`) to automate contract compilation and generate the `blueprint.json` metadata file required for deployment.

```rust
use blueprint_sdk::build;
use blueprint_sdk::tangle::blueprint;
use YOUR_CRATE_LIB_NAME::YOUR_JOB_FUNCTION;
use std::path::Path;
use std::process;

fn main() {
    // Install and update Soldeer for contract building
    let contract_dirs: Vec<&str> = vec!["./contracts"];
    build::utils::soldeer_install();
    build::utils::soldeer_update();
    build::utils::build_contracts(contract_dirs);

    // Invalidate build cache on relevant changes
    println!("cargo:rerun-if-changed=contracts/src");
    println!("cargo:rerun-if-changed=remappings.txt");
    println!("cargo:rerun-if-changed=foundry.toml");
    println!("cargo:rerun-if-changed=../YOUR_CRATE_LIB_NAME");

    // Generate blueprint metadata JSON
    let blueprint = blueprint! {
        name: "YOUR_BLUEPRINT_NAME",
        master_manager_revision: "Latest",
        manager: { Evm = "YOUR_CONTRACT_NAME" },
        jobs: [YOUR_JOB_FUNCTION]
    };

    match blueprint {
        Ok(bp) => {
            let json = serde_json::to_string_pretty(&bp).unwrap();
            std::fs::write(
                Path::new(env!("CARGO_WORKSPACE_DIR")).join("blueprint.json"),
                json.as_bytes(),
            )
            .unwrap();
        }
        Err(e) => {
            eprintln!("cargo:error={:?}", e);
            std::process::exit(1);
        }
    }
}
```

**Notes:**
- Replace `YOUR_CRATE_LIB_NAME`, `YOUR_JOB_FUNCTION`, `YOUR_BLUEPRINT_NAME`, and `YOUR_CONTRACT_NAME` with actual crate, job, blueprint, and contract names.
    - Most blueprint will have multiple Jobs - in that case, all jobs should be listed in the jobs array.
- Ensure `serde_json` is included as a build-dependency in your `Cargo.toml`:


Subsequently, the Cargo.toml located in the {{name}}-bin/ directory should have the following build dependencies:

```toml
[build-dependencies]
blueprint-sdk = { version = "0.1.0-alpha.7", features = ["tangle", "macros"] }
serde_json = "1.0"
```
