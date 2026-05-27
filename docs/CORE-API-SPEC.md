# Core API Specification

## Design Principles

1. **Capability-based access:** Callers possess capabilities, not broad permissions.
2. **Preview before execute:** Every state-changing operation requires a preview step.
3. **Immutable audit:** Every action is logged append-only.
4. **Zero network in crypto layer:** Keyring never talks to the network.

## Module Boundaries

```
crates/
├── types/      # DTOs, no logic
├── crypto/     # Primitives only (AES, Argon2, secp256k1)
├── keyring/    # Keys, seed, derivation — no network
├── provider/   # RPC, Alchemy — no keys
├── router/     # Transaction construction and broadcast
├── txguard/    # Rule engine + Alchemy simulate
├── sign/       # EIP-191, EIP-712, generic tx signing
├── audit/      # SQLite append-only log
└── uniffi/     # FFI bridge — thin wrapper
```

## Public API Surface

### Context (read-only)

```rust
pub async fn wallet_context() -> Result<WalletContext, CoreError>;
pub async fn token_balances(chain_id: Option<u64>) -> Result<Vec<TokenBalance>, CoreError>;
pub async fn transaction_history(
    chain_id: Option<u64>,
    limit: usize,
) -> Result<Vec<Transaction>, CoreError>;
pub async fn positions() -> Result<Vec<Position>, CoreError>;
```

### Preview (simulation)

```rust
pub async fn preview_send(
    to: Address,
    amount: U256,
    chain_id: u64,
) -> Result<SendPreview, CoreError>;

pub async fn preview_swap(
    params: SwapParams,
) -> Result<SwapPreview, CoreError>;

pub async fn preview_stake(
    params: StakeParams,
) -> Result<StakePreview, CoreError>;
```

### Execute (state-changing)

```rust
pub async fn execute_send(
    preview_id: Uuid,
) -> Result<TxHash, CoreError>;

pub async fn execute_swap(
    preview_id: Uuid,
) -> Result<TxHash, CoreError>;

pub async fn execute_stake(
    preview_id: Uuid,
) -> Result<TxHash, CoreError>;
```

### Signing (for WalletConnect / external)

```rust
pub fn sign_message(message: &[u8]) -> Result<Signature, CoreError>;
pub fn sign_typed_data(
    domain_separator: &B256,
    struct_hash: &B256,
) -> Result<Signature, CoreError>;
```

## Error taxonomy

```rust
pub enum CoreError {
    InvalidInput(&'static str),
    Keyring(KeyringError),
    Provider(RpcError),
    TxGuard(Blocked { reason: String }),
    Policy(PolicyError),
    Audit(AuditError),
    PreviewExpired,
    PreviewMismatch,
    InsufficientFunds,
    WalletLocked,
}
```

## gRPC/REST Mapping (future public API)

| Rust fn | gRPC | REST |
|---------|------|------|
| `wallet_context` | `GetWalletContext` | `GET /v1/context` |
| `preview_send` | `PreviewSend` | `POST /v1/preview/send` |
| `execute_send` | `ExecuteSend` | `POST /v1/execute/send` |
