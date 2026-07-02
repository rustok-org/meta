# Core API Specification

> ⚠️ **Частично устарело — до разворота 2026-06-25 (custodial-эра).** Части про «ключ/подпись
> на сервере» неактуальны: продукт развёрнут на non-custodial device-signing. Текущее состояние —
> `PROJECT-STATUS.md`; незыблемое — `core/.claude/NORTH-STAR.md`. Держится как исторический контекст.

> **Status:** Draft — reflects current Rust crate APIs (Phase 1 complete)
> **Target:** Mobile (FFI), LLM Brain (gRPC/REST), MCP (JSON-RPC)
> **Date:** 2026-05-30 (updated 2026-05-31)
> **Edition:** Rust 2024, MSRV 1.91

## Design Principles

1. **Capability-based access:** Callers possess capabilities, not broad permissions.
2. **Preview before execute:** Every state-changing operation requires a preview step.
3. **Immutable audit:** Every action is logged append-only.
4. **Zero network in crypto layer:** Keyring never talks to the network.

---

## Module Boundaries

```
crates/
├── types/      # DTOs, no logic, no secrets
├── crypto/     # Primitives only (AES, Argon2, secp256k1)
├── keyring/    # Keys, seed, derivation — no network
├── provider/   # RPC, Alchemy — no keys
├── router/     # Transaction construction and broadcast
├── txguard/    # Rule engine + Alchemy simulate
├── sign/       # EIP-191, EIP-712, generic tx signing
├── audit/      # SQLite append-only log
└── uniffi/     # FFI bridge — thin wrapper (placeholder)
```

**Trust boundaries:**
- `types` — DTO only, no secrets, safe to serialize over FFI/gRPC
- `crypto` / `keyring` — secrets never cross boundary without encryption
- `sign` — abstraction layer; concrete signing happens in `crypto`
- `audit` — append-only, no delete/update API

---

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

---

## Crate Details

### `types` — Shared DTOs

**Key types:**
```rust
pub struct WalletContext { pub address: Address, pub balances: Vec<ChainBalance>, pub allowed_chains: Vec<u64> }
pub struct ChainBalance { pub chain_id: u64, pub symbol: Box<str>, pub balance: U256 }
pub struct SendPreview { pub preview_id: Uuid, pub to: Address, pub amount: U256, pub chain_id: u64, pub nonce: u64, pub estimated_gas: u64, pub risk_level: RiskLevel }
pub struct SimulationResult { pub gas_used: u64, pub reverted: bool, pub changes: Vec<AssetChange> }

pub enum AuditAction { SignMessage, SignTypedData, SignLegacy, SignEip1559, PreviewBlocked }
pub enum AuditStatus { Success, Failed, Blocked }
pub struct AuditEvent { pub timestamp: u64, pub action: AuditAction, pub address: Address, pub chain_id: Option<u64>, pub preview_id: Option<Uuid>, pub tx_hash: Option<B256>, pub status: AuditStatus, pub error_reason: Option<String>, pub metadata: Option<String> }
```

**Serialization:** All public types implement `Serialize` + `Deserialize`. Enums use `snake_case` renaming.

### `crypto` — Cryptographic Primitives

**Key exports:**
```rust
pub type PrivateKey = Zeroizing<[u8; 32]>;
pub struct RecoverableSignature { pub bytes: [u8; 65] }

pub fn derive_ethereum_key(seed: &Seed) -> Result<PrivateKey, CryptoError>;
pub fn address_from_key(private_key: &PrivateKey) -> Result<Address, CryptoError>;
pub fn sign_digest(digest: &B256, private_key: &PrivateKey) -> Result<RecoverableSignature, CryptoError>;
pub fn sign_message(message: &[u8], private_key: &PrivateKey) -> Result<RecoverableSignature, CryptoError>;
pub fn sign_typed_data(domain_separator: &B256, struct_hash: &B256, private_key: &PrivateKey) -> Result<RecoverableSignature, CryptoError>;
```

**Security invariants:**
- `Seed` does NOT derive `Debug` or `PartialEq` (timing side-channel protection)
- All ECDSA signatures are low-s normalized (`EIP-2`)
- `PrivateKey` is `Zeroizing<[u8; 32]>` — wiped on drop

### `keyring` — Encrypted Keystore

**Key exports:**
```rust
pub struct LocalKeyring;
pub struct ExportedKeyring;
pub struct KeyInfo { pub address: Address, pub label: String }
pub enum KeyringError { WrongPassword, KeyGen(String), Crypto(String), Keystore(String) }

impl LocalKeyring {
    pub fn create(private_key: &PrivateKey, password: &SecretString) -> Result<Self, KeyringError>;
    pub fn unlock(&self, password: &SecretString) -> Result<PrivateKey, KeyringError>;
    pub fn to_signer(&self, password: &SecretString) -> Result<LocalSigner, KeyringError>;
    pub fn export(&self) -> ExportedKeyring;
    pub fn import(exported: &ExportedKeyring, password: &SecretString) -> Result<Self, KeyringError>;
}
```

**Security:** Argon2id (64 MiB, 3 iters, 1 parallelism) + AES-256-GCM with `zeroize`.

### `provider` — Multi-Provider RPC

**Key exports:**
```rust
pub trait Provider: Send + Sync {
    async fn get_balance(&self, address: Address) -> Result<U256, ProviderError>;
    async fn get_nonce(&self, address: Address) -> Result<u64, ProviderError>;
    async fn estimate_gas(&self, tx: &TransactionRequest) -> Result<u128, ProviderError>;
    async fn send_raw_transaction(&self, signed_tx: &Bytes) -> Result<B256, ProviderError>;
    async fn get_transaction_receipt(&self, tx_hash: B256) -> Result<Option<TransactionReceipt>, ProviderError>;
    async fn call(&self, request: &TransactionRequest) -> Result<Bytes, ProviderError>;
    async fn get_fee_data(&self) -> Result<FeeData, ProviderError>;
}

pub struct AlchemyProvider;
pub struct InfuraProvider;
pub struct PublicRpcProvider;
pub enum AnyProvider;
pub struct MultiProvider;
```

**Error variants:** `RateLimited`, `Timeout`, `InvalidRequest(String)`, `ChainNotFound { chain_id }`, `AllEndpointsFailed { chain_id }`, `Rpc(String)` (masked).

### `router` — Transaction Pipeline

**Key exports:**
```rust
pub struct TransactionBuilder;
pub async fn preview<P: Provider>(provider: &P, builder: &TransactionBuilder, from: Address, rules: &RulesEngine, audit: Option<&dyn AuditLog>) -> Result<SendPreview, RouterError>;
pub async fn execute<P: Provider>(provider: &P, preview: &SendPreview, builder: &TransactionBuilder, signer: &dyn Signer, audit: Option<&dyn AuditLog>) -> Result<B256, RouterError>;
```

### `audit` — Append-Only Action Log

```rust
pub trait AuditLog: Send + Sync {
    fn log(&self, entry: &AuditEvent) -> Result<(), AuditLogError>;
}
pub struct SqliteAuditLog;
pub enum AuditLogError { Database, SchemaInit, Closed }
```

---

## Error Taxonomy

```rust
pub enum CoreError {
    InvalidInput(&'static str),
    Keyring(KeyringError),
    Provider(ProviderError),
    TxGuard(Blocked { reason: String }),
    Policy(PolicyError),
    Audit(AuditError),
    PreviewExpired,
    PreviewMismatch,
    InsufficientFunds,
    WalletLocked,
}
```

---

## Security Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│  External (MCP, Mobile, LLM)                                │
│  → FFI / gRPC / JSON                                        │
├─────────────────────────────────────────────────────────────┤
│  Types (DTO) — no secrets, serde only                       │
├─────────────────────────────────────────────────────────────┤
│  Router / TxGuard / Provider — logic only, no secrets       │
├─────────────────────────────────────────────────────────────┤
│  Sign — abstraction, delegates to crypto                    │
├─────────────────────────────────────────────────────────────┤
│  Crypto — secrets in memory, zeroized on drop              │
│  Keyring — secrets encrypted at rest, decrypted on unlock   │
└─────────────────────────────────────────────────────────────┘
```

**Rules:**
1. Secrets never cross the `types` boundary in plaintext
2. `PrivateKey`, `Seed`, `Phrase` are never serialized to JSON or FFI
3. Audit log contains no secret material (only public addresses, tx hashes, timestamps)
4. All errors from crypto/keyring are masked before reaching `types`

---

## FFI / Uniffi Roadmap (Mobile)

**Status:** `crates/uniffi/` is a placeholder — no `Cargo.toml` yet.

**Target consumers:** React Native via `react-native-rust-bridge` or `uniffi-bindgen-react-native`.

**Exposed surface (planned):**

| Rust Type | FFI Type | Direction |
|-----------|----------|-----------|
| `WalletContext` | JSON string | out |
| `SendPreview` | JSON string | out |
| `AuditEvent` | JSON string | out |
| `Address` | Hex string (`0x...`) | in/out |
| `B256` (tx hash) | Hex string | in/out |
| `U256` | Decimal string | in/out |
| `Uuid` | String | in/out |

**Functions (planned):**
```rust
fn wallet_create() -> Result<String, CoreError>;  // Returns mnemonic phrase
fn wallet_import(mnemonic: String) -> Result<Address, CoreError>;
fn wallet_unlock(mnemonic: String, password: String) -> Result<(), CoreError>;
fn wallet_balance(address: String, chain_id: u64) -> Result<String, CoreError>;
fn wallet_nonce(address: String, chain_id: u64) -> Result<u64, CoreError>;
fn tx_preview(request_json: String) -> Result<String, CoreError>;
fn tx_execute(preview_json: String, password: String) -> Result<String, CoreError>;
fn audit_query() -> Result<String, CoreError>;
```

**Security rule for FFI:** Secrets are **never** returned over FFI. Passwords are passed as `secrecy::SecretString` and zeroized immediately.

---

## gRPC/REST Mapping (future public API)

| Rust fn | gRPC | REST |
|---------|------|------|
| `wallet_context` | `GetWalletContext` | `GET /v1/context` |
| `token_balances` | `GetTokenBalances` | `GET /v1/balances` |
| `transaction_history` | `GetTransactionHistory` | `GET /v1/history` |
| `preview_send` | `PreviewSend` | `POST /v1/preview/send` |
| `preview_swap` | `PreviewSwap` | `POST /v1/preview/swap` |
| `preview_stake` | `PreviewStake` | `POST /v1/preview/stake` |
| `execute_send` | `ExecuteSend` | `POST /v1/execute/send` |
| `execute_swap` | `ExecuteSwap` | `POST /v1/execute/swap` |
| `execute_stake` | `ExecuteStake` | `POST /v1/execute/stake` |
| `sign_message` | `SignMessage` | `POST /v1/sign/message` |
| `sign_typed_data` | `SignTypedData` | `POST /v1/sign/typed_data` |
| `audit_query` | `GetAuditLog` | `GET /v1/audit` |

---

## Versioning & Compatibility

- Current version: `0.1.0` (workspace)
- DTO golden tests guarantee backward compatibility for serialization
- Changing a golden test = breaking change = security review required
