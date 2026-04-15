<div align="center">
  <img src="https://raw.githubusercontent.com/tangle-network/tangle/main/assets/Tangle%20%20Banner.png" alt="Tangle Network Banner" width="100%" />
</div>

# Awesome Tangle Blueprints

The canonical catalog of [Tangle Blueprints](https://docs.tangle.tools/developers/blueprints/introduction) — executable service templates that operators run on Tangle Network for fees. Built with [Blueprint SDK](https://github.com/tangle-network/blueprint).

## AI Inference

Operators serve model inference across modalities. Callers pay via x402 HTTP (fast, private) or on-chain jobs (verifiable).

| Blueprint | Modality | Endpoint | SDK Status |
|---|---|---|---|
| [LLM Inference](https://github.com/tangle-network/llm-inference-blueprint) | Text LLM via vLLM | `/v1/chat/completions` | Current |
| [Voice Inference](https://github.com/tangle-network/voice-inference-blueprint) | TTS via Qwen3-TTS / vLLM-Omni | `/v1/audio/speech` | Current |
| [Video Generation](https://github.com/tangle-network/video-gen-inference-blueprint) | Video via ComfyUI / Modal / Replicate | `/v1/video/generate` | Current |
| [Avatar Inference](https://github.com/tangle-network/avatar-inference-blueprint) | Talking-head (HeyGen / D-ID / ComfyUI) | `/v1/avatar/generate` | Current |
| [Embedding](https://github.com/tangle-network/embedding-inference-blueprint) | Text embeddings via HuggingFace TEI | `/v1/embeddings` | Current |
| [Vector Store](https://github.com/tangle-network/vector-store-blueprint) | Qdrant vector DB + similarity search | `/v1/collections/*` | Current |
| [Training](https://github.com/tangle-network/training-blueprint) | Distributed training via DeMo (10,000x compression) | On-chain jobs | Current |
| [Modal Inference](https://github.com/tangle-network/modal-inference-blueprint) | Modal-backed inference proxy | `/v1/chat/completions` | Current |

## AI Agents

Operators host agent runtimes. Callers run AI workloads in isolated environments.

| Blueprint | Description | SDK Status |
|---|---|---|
| [AI Agent Sandbox](https://github.com/tangle-network/ai-agent-sandbox-blueprint) | TEE Docker sandbox runtime (AWS Nitro / Azure / GCP / Phala) — 3 modes: Cloud, Instance, TEE Instance | Current |
| [AI Trading](https://github.com/tangle-network/ai-trading-blueprint) | Autonomous trading with 2-of-3 validator committee, 8 DeFi protocols, ERC-7575 vaults | Current |
| [Secure Code Execution](https://github.com/tangle-network/secure-code-execution-blueprint) | Sandboxed code execution service (Rust, Go, Python, JS/TS) | Current |
| [Rig AI Agent](https://github.com/tangle-network/rig-ai-agent-blueprint) | Rig AI agent with GitHub spell-check/contribution actions | Legacy SDK |
| [Coinbase Agent Kit](https://github.com/tangle-network/coinbase-agent-kit-blueprint) | Coinbase Agent Kit as an operator service | Legacy SDK |
| [Discord Summarizer](https://github.com/tangle-network/discord-summarizer-rig-blueprint) | Discord channel AI summarization agent | Legacy SDK |
| [Gaia AI Agent Template](https://github.com/tangle-network/gaia-ai-agent-template) | Template for AI agents using the Gaia framework | — |

## MPC & Cryptographic Primitives

Operators run threshold cryptography and multi-party computation services.

| Blueprint | Primitive | SDK Status |
|---|---|---|
| [BLS](https://github.com/tangle-network/bls-blueprint) | Threshold BLS signatures (BLS381) | Current |
| [FROST](https://github.com/tangle-network/frost-blueprint) | FROST Schnorr threshold signatures | Current |
| [Schnorr Musig2](https://github.com/tangle-network/schnorr-musig2-blueprint) | Musig2 Schnorr signatures over Secp256k1 | Current |
| [BTE](https://github.com/tangle-network/bte-blueprint) | Batch threshold decryption (MEV-resistant orderflow) | Legacy SDK |
| [DFNS CGGMP21](https://github.com/tangle-network/dfns-cggmp21-blueprint) | CGGMP21 distributed key generation + signing | Legacy SDK |
| [Silent Timelock Encryption](https://github.com/tangle-network/silent-timelock-encryption-blueprint) | Silent setup timelock encryption + threshold decryption | Legacy SDK |
| [WSTS](https://github.com/tangle-network/wsts-blueprint) | Weighted Schnorr threshold signatures | Legacy SDK |
| [CoSnarks zkSaaS](https://github.com/tangle-network/cosnarks-zksaas-blueprint) | Zero-knowledge proofs as a service | Legacy SDK |

## Bridges

| Blueprint | Description |
|---|---|
| [LayerZero DVN Template](https://github.com/tangle-network/layerzero-dvn-blueprint-template) | LayerZero Decentralized Verifier Network template |
| [Hyperlane Relayer](https://github.com/tangle-network/hyperlane-relayer-blueprint) | Hyperlane relayer functionality |
| [Hyperlane ISM Template](https://github.com/tangle-network/hyperlane-ism-blueprint-template) | Hyperlane Interchain Security Module template |
| [Hyperbridge Relayer](https://github.com/tangle-network/hyperbridge-relayer-blueprint) | Hyperbridge relayer functionality |

## Distributed Validators

| Blueprint | Description |
|---|---|
| [Obol DVT](https://github.com/tangle-network/obol-dvt-blueprint) | Obol Distributed Validator Technology |

## Web Services

| Blueprint | Description |
|---|---|
| [TLS Notary](https://github.com/tangle-network/tls-notary-blueprint) | TLS notary services |
| [TX Relayer](https://github.com/tangle-network/txrelayer-blueprint) | Feeless transaction relayer for any EVM (account abstraction) |

## Restaking Integrations

| Blueprint | Description |
|---|---|
| [Incredible Squaring (EigenLayer)](https://github.com/tangle-network/gadget/tree/main/blueprints/incredible-squaring-eigenlayer) | EigenLayer example blueprint |
| [AVS](https://github.com/tangle-network/avs) | Tangle Operator AVS implementation for EigenLayer |
| [Incredible Squaring (Symbiotic)](https://github.com/tangle-network/gadget/tree/main/blueprints/incredible-squaring-symbiotic) | Symbiotic example blueprint |

## Templates & Tools

| Resource | Description |
|---|---|
| [Blueprint SDK](https://github.com/tangle-network/blueprint) | The Rust authoring SDK for all Blueprints (`blueprint-sdk` on crates.io) |
| [Blueprint Template](https://github.com/tangle-network/blueprint-template) | Starter template for new Blueprints |
| [TypeScript Blueprint Template](https://github.com/tangle-network/typescript-blueprint-template) | Template for TypeScript-authored Blueprints |
| [Turnkey API Template](https://github.com/tangle-network/turnkey-blueprint-template) | Template with Turnkey APIs for key management |

## SDK Status

All Blueprints are built with [Blueprint SDK](https://github.com/tangle-network/blueprint) (`blueprint-sdk` crate on crates.io).

| Status | Meaning |
|---|---|
| **Current** | Uses `tangle-network/blueprint` (current SDK repo). Latest published: `0.2.0-alpha.2`. |
| **Legacy SDK** | Uses `tangle-network/gadget` or `tangle-network/gadget-workspace` (predecessor repo). Upgrade to current SDK planned. |

## What is a Tangle Blueprint?

A Blueprint is a Rust service built on Blueprint SDK, paired with a Solidity **BSM** (Blueprint State Machine) contract. The BSM governs operator registration, pricing, slashing, and result verification. Operators register against the BSM, run the service, and earn fees through the [Tangle Protocol payment split](https://docs.tangle.tools):

| Party | Share |
|---|---|
| Developer (Blueprint author) | 20% |
| Protocol treasury | 20% |
| Operators | 40% |
| Stakers | 20% |

Two payment rails per Blueprint:
- **On-chain jobs** via TangleProducer — verifiable results on Tangle
- **x402 HTTP** — fast, private, via the [Shielded Payment Gateway](https://github.com/tangle-network/shielded-payment-gateway)

For documentation: [docs.tangle.tools/developers/blueprints/introduction](https://docs.tangle.tools/developers/blueprints/introduction)

## Contributing

We welcome contributions. To add a Blueprint, open a PR with a link to the repo and a one-line description.
