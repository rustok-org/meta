# Session: Core Audit Remediation

> **Дата:** 2026-05-28  
> **Scope:** `core/crates/{types, crypto, keyring}`  
> **Базис:** Объединённый аудит Codex + Security Team  
> **Владелец:** @temrjan  
> **Статус:** Step 1 / 5  

---

## Workflow (обязателен для каждого шага)

**Строго последовательно. Никакой параллели.**

```
┌─────────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────┐   ┌──────────────┐   ┌──────┐
│ Реализация  │ → │ Commit       │ → │ Code Review   │ → │ Fix  │ → │ Security Rev │ → │ Fix  │
│ (local dev) │   │ (conventional│   │ (self/peer)   │   │      │   │ (checklist)  │   │      │
└─────────────┘   └──────────────┘   └───────────────┘   └──────┘   └──────────────┘   └──────┘
                                                                                                 
┌──────────┐   ┌──────────────┐   ┌──────────┐
│ Push     │ → │ CI green     │ → │ Merge    │
│ (только  │   │ (cargo test, │   │ (squash) │
│ после 2x │   │ clippy, deny)│   │          │
│ review ✅│   └──────────────┘   └──────────┘
└──────────┘
```

### Запрещено (hard rules)

- [ ] **Push без Code Review** — запрещено.
- [ ] **Push без Security Review** — запрещено.
- [ ] **Merge с красным CI** — запрещено.
- [ ] **Параллельные PR** — запрещено. Следующий шаг начинаем только после `merge` предыдущего.
- [ ] **Прямые коммиты в `main`** — запрещено. Только PR + squash merge.
- [ ] **Skip local checks** — запрещено. Перед коммитом: `cargo test`, `cargo clippy`, `cargo fmt --all --check`, `cargo deny`, `cargo audit`.

---

## Roadmap (последовательность)

| Step | PR | Название | Крейт | Приоритет | Зависимость |
|------|----|----------|-------|-----------|-------------|
| 1 | `#1` | `[SECURITY] Crypto signing hardening` | `crypto` | 🔴 Critical | — |
| 2 | `#2` | `[SECURITY] Keyring memory & error hygiene` | `keyring` | 🔴 Critical | Step 1 ✅ |
| 3 | `#3` | `[ARCH] Types error model & DTO hardening` | `types` | 🟡 High | Step 2 ✅ |
| 4 | `#4` | `[DEPS] Dependency audit & crypto API` | `crypto` | 🟡 Medium | Step 3 ✅ |
| 5 | `#5` | `[UX] Keyring metadata design` | `keyring` | 🟢 Low | Step 4 ✅ |

> **Rule:** Следующий Step открывается только после `git merge --squash` предыдущего и зелёного CI на `main`.

---

## Step 1 — PR #1: `[SECURITY] Crypto signing hardening`

### Scope
- `core/crates/crypto/src/sign.rs`
- `core/crates/crypto/src/seed.rs`
- `core/crates/crypto/Cargo.toml`

### Задачи

#### 1.1 Low-s normalization (C1)
**Файл:** `crypto/src/sign.rs`  
**Строка:** после `sign_recoverable` в `sign_digest`.

```rust
let (sig, recid) = signing_key
    .sign_recoverable(digest.as_slice())
    .map_err(|e| CryptoError::SigningFailed(e.to_string()))?;

// FIX: добавить нормализацию s для защиты от malleability (EIP-2)
let sig = sig.normalize_s().unwrap_or(sig);
```

**Критерий:** `sign_message` → `sign_digest` должен возвращать подпись с `s <= n/2`.  
**Тест:** golden test с известным мнемоником + сообщение → hex подписи не меняется (determinism).

#### 1.2 Seed: убрать `PartialEq, Eq` (C3)
**Файл:** `crypto/src/seed.rs`  
**Строка:** `#[derive(Debug, Clone, PartialEq, Eq)]` → `#[derive(Debug, Clone)]`.

**Критерий:** `Seed` не реализует `PartialEq`.  
**Тест:** заменить `assert_eq!(seed1.as_bytes(), seed2.as_bytes())` на `assert!(subtle::ConstantTimeEq::ct_eq(...).into())`.  
**Dev-dep:** добавить `subtle = "2.6"` в `[dev-dependencies]`.

#### 1.3 `expect` → `unwrap` (L2)
**Файл:** `crypto/src/sign.rs`  
**Строки:** 31, 35, 133.

```rust
// Было:
.try_into().expect("exactly 32 bytes")
// Стало:
.try_into().unwrap()
```

**Критерий:** `cargo clippy` чист, нет `expect`.

#### 1.4 `sign_message`: `format!` → `Vec<u8>` (L1)
**Файл:** `crypto/src/sign.rs`  
**Строка:** `let prefix = format!(...);`.

```rust
// Было:
let prefix = format!("\x19Ethereum Signed Message:\n{}", message.len());
let mut full_message = prefix.into_bytes();

// Стало:
let prefix = format!("\x19Ethereum Signed Message:\n{}", message.len());
let mut full_message = Vec::with_capacity(prefix.len() + message.len());
full_message.extend_from_slice(prefix.as_bytes());
```

**Критерий:** `sign_message` работает, тесты проходят.

### Gates (перед коммитом)

```bash
cd core
cargo test -p rustok-crypto
cargo clippy -p rustok-crypto --all-targets --all-features
cargo fmt --all --check
cargo deny check
cargo audit
```

### Commit message
```
fix(crypto): low-s normalization, Seed timing, idioms

- Normalize s in ECDSA signature (EIP-2 malleability fix)
- Remove PartialEq/Eq from Seed to prevent timing leak
- Use Vec::with_capacity instead of format!+into_bytes
- Replace expect with unwrap in sign.rs

Security-Audit: C1, C3, L1, L2
```

---

## Security Review Checklist (для каждого шага)

- [ ] **No new secrets in code** — проверить diff на `println!`, `dbg!`, hardcoded keys.
- [ ] **Zeroize hygiene** — все новые `[u8; N]` / `Vec<u8>` с секретами обёрнуты в `Zeroizing`.
- [ ] **Error masking** — crypto-путь не прокидывает `e.to_string()` наружу.
- [ ] **No timing side-channels** — нет `PartialEq` / `==` на секретных данных.
- [ ] **No new `unsafe`** — `#![forbid(unsafe_code)]` на уровне workspace.
- [ ] **Determinism** — golden tests не сломаны.
- [ ] **`cargo deny` чист** — нет новых yanked / unlicensed зависимостей.

---

## Code Review Checklist (для каждого шага)

- [ ] **Read before write** — автор прочитал файлы, которые меняет, + 1–2 файла, которые их импортируют.
- [ ] **Minimal diff** — только то, что нужно. Нет «пока тут, заодно...».
- [ ] **Naming** — новые функции/переменные по `snake_case`, типы `PascalCase`.
- [ ] **Docs** — новые `pub` элементы задокументированы (`///`).
- [ ] **Tests** — каждый новый `pub` метод имеет тест (unit или property).
- [ ] **CI green** — `cargo test`, `clippy`, `fmt`, `deny`, `audit` проходят.

---

## Next Steps

- **Сейчас:** передать Step 1 инженеру.
- **После коммита инженера:** я проверяю коммит по чеклистам выше.
- **После Security Review:** approve → push → merge → открыть Step 2.
