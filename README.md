# D3FAULT Encryption Method

> Commit-reveal privacy protocol for private asset transfers on Solana mainnet.

**Program ID:** `2akvmbaGhGeAWVCQrpHXvJTZE9EksMk5rpHM4NcMmfKG`  
**Network:** Solana mainnet-beta  
**Anchor:** 0.29.0 · Non-upgradeable · [Verified source](https://github.com/d3faultlabs/d3fault-program)

---

## Overview

D3FAULT uses a **commit-reveal scheme** to break the on-chain link between a depositor and a recipient. No trusted setup. No zero-knowledge circuits. No intermediary custody. Just cryptographic hash functions and a non-upgradeable Solana program.

The core invariant: **the secret never touches the blockchain.** Only its SHA-256 hash is stored on-chain. Anyone who knows the secret can claim the funds — to any wallet, at any time, from any device — without revealing who deposited them.

---

## How It Works

### 1. The Commitment

When a user deposits funds, they generate a random 32-byte secret locally:

```
secret    = random_bytes(32)           // never leaves the user's device
commitment = SHA-256(secret)           // only this is stored on-chain
```

The `commitment` is a deterministic, one-way fingerprint of the secret. Knowing the commitment tells an observer nothing about the secret, and knowing the secret instantly reveals which commitment it maps to.

### 2. Deposit (Commit Phase)

The user signs a transaction that:

1. Transfers a protocol fee (0.25% for SOL, flat 0.0021 SOL for SPL tokens) to the relayer
2. Calls `deposit_sol` or `deposit_spl` on the on-chain program

The program stores the following in the `CommitmentStore` PDA:

| Field | Type | Description |
|---|---|---|
| `commitment` | `[u8; 32]` | SHA-256 hash of the secret |
| `amount` | `u64` | Lamports or token amount locked |
| `expiry` | `i64` | Unix timestamp after which depositor can reclaim |
| `token_mint` | `[u8; 32]` | SPL mint address, or zero bytes for native SOL |
| `claimed` | `u8` | 0 = unclaimed, 1 = claimed |
| `depositor` | `[u8; 32]` | Original depositor pubkey (for reclaim only) |

**Nothing in the deposit transaction references the recipient.** The recipient is chosen at claim time.

### 3. Withdrawal (Reveal Phase)

To withdraw, the recipient submits the secret (off-chain, to the relayer API). The relayer:

1. Recomputes `commitment = SHA-256(secret)`
2. Locates the matching entry in the `CommitmentStore`
3. Verifies the commitment is unclaimed and not expired
4. Signs and submits a `withdraw_sol` or `withdraw_spl` transaction

The on-chain program:

1. Recomputes `SHA-256(secret)` and checks it matches the stored commitment
2. Initialises a `NullifierRecord` PDA — derived from the commitment — to prevent double-spending
3. Transfers funds directly to the recipient wallet

The secret is passed as a transaction instruction argument and is never stored anywhere.

---

## On-Chain Architecture

### CommitmentStore PDA

```
seeds = ["commitment_store"]
size  = 8 + 16 + (64 × 120) = 7,832 bytes
```

A fixed circular buffer of 64 commitment slots using Anchor's `zero_copy` attribute. Each slot is 120 bytes:

```
commitment : [u8; 32]   // SHA-256 hash
amount     : u64        // 8 bytes
expiry     : i64        // 8 bytes
token_mint : [u8; 32]   // 32 bytes (zero = native SOL)
claimed    : u8         // 1 byte
_pad       : [u8; 7]    // alignment padding
depositor  : [u8; 32]   // 32 bytes
```

The store is intentionally fixed at 64 slots. Slots are reused in a ring-buffer pattern once claimed or expired. This bounds on-chain storage to a constant size regardless of protocol usage.

### NullifierRecord PDA

```
seeds = ["nullifier", commitment_bytes]
```

Initialised at withdrawal time. Its existence proves a commitment has been spent. The program rejects any withdrawal attempt if this PDA already exists, making double-spending a hard on-chain impossibility — not a soft check.

### Program Instructions

| Instruction | Description |
|---|---|
| `deposit_sol` | Lock native SOL against a commitment hash |
| `deposit_spl` | Lock SPL tokens against a commitment hash |
| `withdraw_sol` | Reveal secret, verify commitment, release SOL |
| `withdraw_spl` | Reveal secret, verify commitment, release SPL tokens |
| `reclaim_sol` | Depositor reclaims funds after expiry (no secret needed) |

---

## Privacy Model

### Why a Relayer?

Without a relayer, the recipient must sign the withdrawal transaction — which links their wallet to the withdrawal on-chain. With a relayer, the recipient sends the secret off-chain over HTTPS, and the relayer submits the transaction on their behalf.

### Fresh Ephemeral Relayer (Per-User Isolation)

D3FAULT does not use a single shared relayer wallet for all withdrawals. Instead, every withdrawal uses a **fresh ephemeral keypair**:

```
prepare-withdraw  →  server generates fresh keypair
                  →  server funds it (main_relayer → fresh_relayer)
                  →  returns token + fresh_relayer pubkey to user

relay-withdraw    →  user provides secret + prepare token
                  →  fresh relayer signs and submits withdraw tx
                  →  leftover SOL swept back to main relayer
```

**The result:**

| On-chain event | Visible parties |
|---|---|
| `main_relayer → fresh_relayer` | Server-side only. No user wallet. |
| `user → CommitmentStore` | Depositor + program. No fresh relayer. |
| `fresh_relayer → recipient` | Fresh relayer + recipient. No depositor. |

There is no single transaction or account that links all three. An on-chain observer sees three disconnected events. The connection between them exists only on the server — and even there it is ephemeral (15-minute TTL, then discarded).

### What Is and Is Not Hidden

| Hidden | Not Hidden |
|---|---|
| Link between depositor and recipient | That a deposit occurred |
| Recipient wallet at deposit time | Deposited amount |
| Secret (never on-chain) | Commitment hash (intentionally public) |
| Fresh relayer identity | That a withdrawal occurred |

This is not a full anonymity set (like Tornado Cash). It is a **sender-recipient unlinkability** scheme. The depositor and recipient are decoupled unless the secret is shared through a traceable channel.

---

## Why Solana

### Accounts Model

Solana's accounts model allows the `CommitmentStore` to be a single fixed-size PDA shared by all users. On EVM chains, achieving equivalent behaviour requires either a mapping (unbounded storage growth) or a Merkle tree (additional circuit complexity). On Solana, the entire active commitment buffer is 7.8 KB and always rent-exempt.

### Nullifiers as PDAs

Solana PDA existence checks are O(1) and free. On EVM, checking a nullifier requires a `SLOAD` opcode (cold read: 2,100 gas). Solana's approach is structurally equivalent to ZK nullifier sets but without any proof system — just a PDA derivation and an existence check.

### Non-Upgradeable Deployment

The program is deployed as non-upgradeable. No upgrade authority exists. No admin can modify withdrawal logic, redirect funds, or change fee parameters after deployment. The deployed bytecode is what it is — auditable and immutable.

### SPL Token Support

Native multi-token support with no additional abstraction layer. Depositing USDC, JitoSOL, or any SPL token follows the same commit-reveal flow as native SOL, using ATAs (Associated Token Accounts) under the `CommitmentStore` PDA as escrow.

### Throughput

Solana block time is ~400ms. A deposit-to-withdrawal round trip completes in under 2 seconds under normal network conditions — fast enough for a DEX UX without compromising the privacy model.

---

## API Reference

The relayer API is served from `artifacts/api-server` in [d3faultlabs/d3fault-production](https://github.com/d3faultlabs/d3fault-production).

### Core Relay Endpoints

```
POST /prepare-withdraw
  → Generates a fresh ephemeral relayer and funds it
  ← { token, freshRelayerPubkey }

POST /relay-withdraw
  Body: { secret, recipient, prepareToken }
  → Verifies commitment, submits withdraw tx, sweeps relayer
  ← { signature, explorerUrl }
```

### Commitment Store Read Endpoints

```
GET /api/v1/store/commitments?status=all|active|claimed|expired
  ← ParsedEntry[]

GET /api/v1/store/lookup/:commitment
  ← ParsedEntry | 404
```

### Recovery Endpoints

```
GET  /pending-reclaims?depositor=<pubkey>
  ← { reclaims: PendingReclaimEntry[] }

DELETE /pending-reclaims/:commitmentHex
  ← { ok: true }
```

### Utility

```
GET  /price?mint=<mint>          → USD price + 24h change
POST /swap-fund                  → Fund ephemeral wallet for SPL swaps (0.01 SOL)
GET  /api/healthz                → { ok: true, network, programId }
```

---

## Security Properties

| Property | Mechanism |
|---|---|
| Double-spend prevention | NullifierRecord PDA — hard on-chain block |
| Expiry enforcement | `i64` expiry field checked by program at instruction time |
| Commitment collision resistance | SHA-256 (preimage resistance: 2^256 work) |
| Relayer compromise | Fresh relayer holds no funds between sessions; main relayer is not in the withdrawal path |
| Upgrade risk | Non-upgradeable program — no admin authority |
| Front-running | Commitment is submitted first; secret revealed only at withdrawal |

---

## Repository Map

```
d3faultlabs/
├── d3fault-Encryption-method   ← this repo (protocol documentation)
├── d3fault-program             ← Anchor program source (Rust)
└── d3fault-production          ← Full stack (API server, web frontend, status page)
```

---

[d3fault.sh](https://d3fault.sh) · [System Status](https://d3fault.sh/status/) · [MIT License](https://github.com/d3faultlabs/d3fault-production/blob/main/LICENSE)
