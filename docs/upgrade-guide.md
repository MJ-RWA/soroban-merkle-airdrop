# Soroban SDK Upgrade Guide

This guide explains how to upgrade the Soroban SDK version used by the
`soroban-merkle-airdrop` contract, what to watch out for, and what operational
steps are required after an upgrade.

The current SDK version is pinned in the workspace:

```toml
# Cargo.toml (workspace root)
[workspace.dependencies]
soroban-sdk = { version = "27.0.6" }
```

---

## Why upgrading matters

The Soroban SDK is a thin shim over the host ABI. When Stellar upgrades the
network protocol, the host ABI may change in ways that require a new SDK. A
contract compiled against an old SDK may refuse to deploy, or may fail at
runtime, on a network that has moved to a newer protocol version.

**Every SDK upgrade requires new WASM on-chain.** The compiled WASM is what
lives on-chain; updating `Cargo.toml` and rebuilding is not enough on its own —
the new binary must be uploaded. It can then either be applied to the existing
contract with the admin-only `upgrade()` entry point (see
[Contract WASM upgrade](#contract-wasm-upgrade)) or deployed as a new contract
instance.

---

## Step-by-step upgrade procedure

### 1. Update the SDK version in `Cargo.toml`

Open the workspace `Cargo.toml` at the repository root and bump the version:

```toml
[workspace.dependencies]
soroban-sdk = { version = "28.0.0" }   # ← new version
```

The contract's own `Cargo.toml` (`contracts/airdrop/Cargo.toml`) inherits this
via `{ workspace = true }`, so you only need to change one file.

### 2. Update the lock file

```bash
cargo update -p soroban-sdk
```

This resolves the new version and writes an updated `Cargo.lock`. Commit both
`Cargo.toml` and `Cargo.lock` together.

### 3. Rebuild the contract

```bash
cargo build \
  --manifest-path contracts/airdrop/Cargo.toml \
  --target wasm32v1-none \
  --release
```

The output WASM is at:

```
target/wasm32v1-none/release/soroban_merkle_airdrop.wasm
```

### 4. Re-run the test suite

```bash
cargo test --manifest-path contracts/airdrop/Cargo.toml
```

All 25+ tests must pass. Pay special attention to:

- `test_leaf_hash_known_vector` — verifies that the Rust leaf hash still
  produces the same bytes as the TypeScript SDK. If the SDK changes how
  `BytesN`, `Bytes`, or `Address::to_string()` work, this cross-language
  vector test will catch the divergence.
- `test_restore_extends_ttl` — verifies TTL extension behaviour, which depends
  on host semantics.

### 5. Check the TypeScript SDK

If the upgrade involves a change to how contract ABI types are serialised (e.g.
changes to `contracttype`, `BytesN`, or `Address`), also rebuild and re-test
the TypeScript SDK:

```bash
cd sdk && pnpm install && pnpm build && pnpm test
```

The `merkle.test.ts` file contains the same leaf-hash vector. Both Rust and
TypeScript must produce identical bytes.

### 6. Deploy the new WASM

After passing all tests, upload and deploy:

```bash
# Upload the new WASM binary
stellar contract upload \
  --network testnet \
  --source deployer \
  --wasm target/wasm32v1-none/release/soroban_merkle_airdrop.wasm

# Note the new WASM hash printed by the upload command, then deploy
stellar contract deploy \
  --network testnet \
  --source deployer \
  --wasm-hash <NEW_WASM_HASH>
```

The new deployment gets a fresh contract address. Initialise it with a **new
Merkle root** (see the replay-protection note in
[`docs/threat-model.md`](./threat-model.md#9-same-network-re-deployment-replay-attack)).

### 7. Update documentation and deployment records

Record the new contract address, WASM hash, SDK version, and Merkle root in
your deployment log. Update `README.md` and `docs/deploying.md` if the CLI
commands or default addresses have changed.

---

## Contract WASM upgrade

A deployed airdrop can replace its own WASM in place: the admin-only
`upgrade(new_wasm_hash)` entry point in
[`contracts/airdrop/src/lib.rs`](../contracts/airdrop/src/lib.rs) calls
`update_current_contract_wasm`. The contract address, admin, Merkle root, token
address, and all claim state are preserved. This is distinct from the SDK
version procedure above, which describes rebuilding from source; use this
runbook when the goal is to ship a fixed or optimized binary to an **already
deployed** contract.

### 1. When to use

- Bug fixes, gas optimizations, and compatible SDK bumps that keep the same
  contract interface and storage keys.
- Do **not** use it for ABI-breaking changes (renamed or removed functions,
  changed argument or event shapes) or for storage-layout changes. In those
  cases deploy a new contract instance instead, as described in
  [Step 6](#6-deploy-the-new-wasm), and initialize it with a fresh Merkle root
  (see the
  [same-network re-deployment replay note](./threat-model.md#9-same-network-re-deployment-replay-attack)).

### 2. Preparation

- Make sure the current admin key is available and can sign. `upgrade()` calls
  `admin.require_auth()`.
- **Pause the contract first** so claims cannot land during the upgrade
  window:

  ```bash
  stellar contract invoke \
    --network <network> \
    --source admin \
    --id "$CONTRACT_ID" \
    -- set_active --active false
  ```

- Record the current WASM hash and the `contract_info()` output as a rollback
  reference. Keep the old WASM file so the previous hash can be uploaded again
  if needed.

### 3. Build new WASM

```bash
cargo build \
  --manifest-path contracts/airdrop/Cargo.toml \
  --target wasm32v1-none \
  --release
```

Run the test suite (`cargo test --manifest-path contracts/airdrop/Cargo.toml`)
before uploading. The artifact is
`target/wasm32v1-none/release/soroban_merkle_airdrop.wasm`.

### 4. Upload new WASM

```bash
stellar contract upload \
  --network <network> \
  --source admin \
  --wasm target/wasm32v1-none/release/soroban_merkle_airdrop.wasm
```

Capture the `<NEW_WASM_HASH>` printed by the command.

### 5. Execute upgrade

```bash
stellar contract invoke \
  --network <network> \
  --source admin \
  --id "$CONTRACT_ID" \
  -- upgrade --new_wasm_hash <NEW_WASM_HASH>
```

The admin must authorize the call. On success an `("upgrade", admin)` event
with the new hash is emitted, so indexers can detect the change. The upgrade is
immediate — there is no staged upgrade or time-lock.

### 6. Verify

Confirm the new WASM hash is active using an explorer or the contract's ledger
entry, and check that state is unchanged with `contract_info()`:

```bash
stellar contract invoke --network <network> --id "$CONTRACT_ID" -- contract_info
```

`contract_info()` returns `admin`, `token`, `merkle_root`, `total_deposited`,
`active`, and `expiration` — all of these should be unchanged (including
`active: false`). Claimed flags are unchanged too; verify one with
`is_claimed()`. On testnet, exercise a claim from an unclaimed address before
unpausing on mainnet.

### 7. Unpause

```bash
stellar contract invoke \
  --network <network> \
  --source admin \
  --id "$CONTRACT_ID" \
  -- set_active --active true
```

Confirm `is_active()` returns `true` and monitor the first claims and the
`("upgrade", admin)` / `("paused", admin)` events.

### 8. Caveats

- Storage keys must be backward compatible. Every entry written by the old
  WASM must still be readable with the same `DataKey` variants and types;
  claimed entries and the Merkle root must not be lost.
- ABI-breaking changes require redeployment — do not point an in-place upgrade
  at a binary whose function signatures or event topics changed, because
  existing clients and indexers will break.
- The upgrade is immediate and has no time-lock, so pause first and use a
  rehearsal on testnet.
- Verify on testnet before mainnet, and record the new hash, SDK version, and
  timestamp in the deployment log.

---

## Known breaking changes by version

### Upgrading from 20.x to 21.x

- `soroban_sdk::token::Client` was introduced; earlier versions used a different
  token interface. The contract already uses `token::Client`, so no change
  needed here if you are on 20.x or later.
- `contractmeta!` macro syntax was stabilised. No change required.

### Upgrading from 21.x / 22.x to 27.x

- The `wasm32v1-none` target replaced `wasm32-unknown-unknown` for contract
  builds. Update any CI build scripts or local build aliases.
- `Events::publish` was deprecated in 27.x in favour of the `#[contractevent]`
  macro. The contract uses the old form intentionally (see the `#[allow(deprecated)]`
  comment in `lib.rs`). Switching to `#[contractevent]` would rename event
  topics and is a deliberate ABI decision, not a routine upgrade step.
- `Address::from_str` is available from 20.x onwards; no change.
- `env.ledger().set_timestamp()` in tests was stabilised in the `testutils`
  feature by 21.x; no change.

### Upgrading from 27.x to 28.x (anticipated)

No breaking changes to the contract surface have been announced at the time of
writing. Check the [Soroban SDK CHANGELOG](https://github.com/stellar/rs-soroban-sdk/blob/main/CHANGELOG.md)
and the [Stellar Protocol upgrade notes](https://developers.stellar.org/docs/learn/encyclopedia/network-configuration/protocol-upgrades)
before upgrading.

Watch for:

- Changes to `BytesN` or `Bytes` serialisation — would affect `leaf_hash` and
  the cross-language test vector.
- Changes to `contracttype` struct field serialisation order — would affect
  `ContractInfo` and `AirdropEntry`.
- Changes to the `testutils` API — would require updating `test.rs`.

---

## Operational reminder: in-place upgrade vs. new deployment

The contract does expose `update_current_contract_wasm` through the admin-only
`upgrade()` entry point, so a compatible upgrade can be applied in place (see
[Contract WASM upgrade](#contract-wasm-upgrade)). That keeps the contract
address, Merkle root, and claim state intact.

A **new deployment** is still required when the ABI or storage layout changes.
It means:

1. A new contract address.
2. A new Merkle root (required by the replay-protection policy).
3. New token deposits into the new contract.
4. Communication to all recipients about the new contract address.
5. Pausing or retiring the old contract.

Plan breaking upgrades well before mainnet deployment to avoid forcing
recipient re-notification. On testnet, upgrades are cheap — test freely.
