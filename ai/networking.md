# Cursor Rules: Blueprint Networking SDK

This document explains how to use the Blueprint SDK’s networking primitives to integrate libp2p-based peer-to-peer messaging into any Tangle or Eigenlayer Blueprint. It focuses on instantiating the networking layer in production contexts, configuring allowed keys from multiple environments, and composing custom P2P services.

---

## 1. Networking Overview

The Blueprint SDK supports P2P communication via:
- `NetworkService` — manages the network lifecycle
- `NetworkServiceHandle` — used in jobs/contexts to send/receive messages
- `NetworkConfig` — initializes node identity, protocol name, allowed keys
- `AllowedKeys` — limits which nodes can connect

The networking stack is libp2p-native and works in Tangle, Eigenlayer, or custom Blueprint deployments.

---

## 2. Integrating Networking into a Context

### Context Layout
```rust
#[derive(Clone, KeystoreContext)]
pub struct MyContext {
    #[config]
    pub config: BlueprintEnvironment,
    pub network_backend: NetworkServiceHandle,
    pub identity: sp_core::ecdsa::Pair, // or other signing key
}
```

### Context Constructor
```rust
pub async fn new(config: BlueprintEnvironment) -> Result<Self> {
    let allowed_keys = get_allowed_keys(&config).await?;
    let network_config = config.libp2p_network_config("/my/protocol/1.0.0")?;
    let network_backend = config.libp2p_start_network(network_config.clone(), allowed_keys)?;

    Ok(Self {
        config,
        network_backend,
        identity: network_config.instance_key_pair.0.clone(),
    })
}
```

---

## 3. Computing Allowed Keys

### From Tangle
```rust
let operators = config.tangle_client().await?.get_operators().await?;
let allowed_keys = AllowedKeys::InstancePublicKeys(
    operators.values().map(InstanceMsgPublicKey).collect()
);
```

### From Eigenlayer AVS
```rust
let client = EigenlayerClient::new(config.clone());
let (addrs, pubkeys) = client
    .query_existing_registered_operator_pub_keys(start_block, end_block)
    .await?;

let keys = pubkeys
    .into_iter()
    .filter_map(|k| k.bls_public_key)
    .map(|pk| {
        let ark_pk = blueprint_crypto::bn254::ArkBlsBn254::Public::deserialize_compressed(&pk)?;
        InstanceMsgPublicKey::from_bn254(&ark_pk)
    })
    .collect();

let allowed_keys = AllowedKeys::InstancePublicKeys(keys);
```

---

## 4. Sending and Receiving Messages

### Sending
```rust
let routing = MessageRouting {
    message_id: 1,
    round_id: 0,
    sender: ParticipantInfo::from(identity),
    recipient: None, // Gossip
};

context.network_backend.send(routing, message_bytes)?;
```

### Receiving
```rust
if let Some(msg) = context.network_backend.next_protocol_message() {
    // Deserialize and handle
}
```

Use `bincode` or similar for message serialization.

---

## 5. Notes on Identity

- Identity for `NetworkConfig` comes from the `instance_key_pair` field
- The `InstanceMsgPublicKey` must match one used in the `AllowedKeys`
- Supported key types: `SpEcdsa`, `ArkBlsBn254`, others via `KeyType` trait

---

## 6. Best Practices

DO:
- Use context-level networking — never instantiate inside jobs
- Set unique protocol ID per service (`/app/version/...`)
- Use canonical serialization formats

DON’T:
- Use test keys or unverified peer identities in production
- Recreate the network multiple times per job instance

---

## 7. Use Cases
- Gossip consensus messages across validator peers
- Coordinate operator stake verification or rewards
- Build secure MPC jobs across ECDSA/BLS keys
- Trigger tasks from P2P rather than onchain events

---

For round-based coordination, see the `round-based.md` doc.

