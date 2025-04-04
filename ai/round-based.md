# Cursor Rules: Round-Based Protocols with Blueprint SDK

This guide describes how to design and execute round-based multiparty protocols using the `round_based` crate and Blueprint SDK’s `RoundBasedNetworkAdapter`. These protocols are ideal for DKG, randomness generation, keygen, signing, or any interactive consensus.

---

## 1. Key Concepts

- **MpcParty**: Abstraction over a network-connected party
- **RoundsRouter**: Drives round orchestration, ensures all inputs are gathered
- **RoundInput**: Declares message shape and broadcast/point-to-point semantics
- **ProtocolMessage**: Trait to derive on all messages (requires `Serialize`, `Deserialize`)
- **MsgId**: Tracks individual messages for blame

---

## 2. Define Protocol Messages

```rust
#[derive(Clone, Debug, PartialEq, ProtocolMessage, Serialize, Deserialize)]
pub enum Msg {
    Commit(CommitMsg),
    Decommit(DecommitMsg),
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct CommitMsg {
    pub commitment: [u8; 32],
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct DecommitMsg {
    pub randomness: [u8; 32],
}
```

---

## 3. Set Up the Router

```rust
let mut router = RoundsRouter::<Msg>::builder();
let round1 = router.add_round(RoundInput::<CommitMsg>::broadcast(i, n));
let round2 = router.add_round(RoundInput::<DecommitMsg>::broadcast(i, n));
let mut router = router.listen(incoming); // from MpcParty::connected(...)
```

---

## 4. Send and Receive

```rust
outgoing.send(Outgoing::broadcast(Msg::Commit(CommitMsg { ... }))).await?;
let commits = router.complete(round1).await?;
```

You may access indexed results and verify per party.

---

## 5. Connect to Network

```rust
let network = RoundBasedNetworkAdapter::new(
    context.network_backend.clone(),
    local_index,             // your own party index
    indexed_keys,            // PartyIndex → InstanceMsgPublicKey
    "round-protocol-instance-id"
);
let MpcParty { delivery, .. } = MpcParty::connected(network).into_party();
let (incoming, outgoing) = delivery.split();
```

You now have `incoming` and `outgoing` channels to wire into your protocol.

---

## 6. Simulating the Protocol

For local dev:
```rust
round_based::sim::run_with_setup(parties, |i, party, rng| async move {
    protocol_fn(party, i, n, rng).await
})
.expect_ok()
.expect_eq();
```

---

## 7. Production Pattern
Use the adapter in a background task or job with:
- `RoundBasedNetworkAdapter`
- Indexed `InstanceMsgPublicKey`s
- State machine logic coordinating rounds
- Optional blame tracking

---

## 8. Blame Tracking
To identify misbehavior:
```rust
pub struct Blame {
    pub guilty_party: PartyIndex,
    pub commitment_msg: MsgId,
    pub decommitment_msg: MsgId,
}
```

If `commit != sha256(decommit)`, blame the peer and continue protocol.

---

## 9. Error Handling
Use rich error types to pinpoint issues:
```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("failed to send commitment")]
    Round1Send(#[source] SendError),
    #[error("decommitment mismatch")]
    InvalidDecommitment { guilty: Vec<Blame> },
    // ...
}
```

---

## 10. Use Cases
- Randomness beacons
- DKG or key resharing
- Aggregated signing
- Verifiable shuffles
- Voting and consensus schemes

---

Use this guide to scaffold secure, blame-attributing, peer-verifiable round-based protocols.
