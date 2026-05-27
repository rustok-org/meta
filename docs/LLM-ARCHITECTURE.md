# LLM Architecture

## Mental Model

**The Pilot.** Receives natural language, understands intent, selects tools, confirms with user, executes via core.

## Components

### 1. Intent Parser

Input: `"Send Alice 0.5 ETH on Base"`
Output: `Intent::Send { to: "Alice", amount: "0.5", token: "ETH", chain: "Base" }`

- Uses LLM completion for fuzzy parsing.
- Fallback to regex for common patterns.
- Validates against address book and known aliases.

### 2. Entity Resolver

Resolves ambiguous entities:
- "Alice" → `0x1234...5678` (address book lookup)
- "подешевле" → `Slippage::Min`
- "быстро" → `GasPriority::High`

### 3. Memory

**Short-term:** Current dialog context (last 10 messages).  
**Long-term:** User preferences, frequent recipients, risk tolerance.

Storage: SQLite or vector DB (for semantic search over history).

### 4. Tool Registry

Declarative list of available core functions:

```rust
#[derive(Debug)]
pub struct Tool {
    pub name: &'static str,
    pub description: &'static str,
    pub parameters: Vec<Parameter>,
    pub handler: Box<dyn Fn(Json) -> Result<Json, ToolError>>,
}
```

The LLM selects tools based on intent. The executor validates parameters before calling core.

### 5. Dialog Manager

States:
```
Chat → AwaitingConfirmation → Executed
        ↓
   NeedsClarification ←── (ambiguous input)
```

### 6. Risk Explainer

Translates txguard `Verdict` into human language:

| Verdict | Explainer output |
|---------|------------------|
| ALLOW | "✅ Транзакция безопасна" |
| WARN | "⚠️ Контракт создан 2 дня назад. Продолжить?" |
| BLOCK | "❌ Заблокировано: unlimited approval" |

## Data Flow

```
User text
   ↓
Intent Parser → Intent
   ↓
Entity Resolver → ResolvedIntent
   ↓
Tool Selector → Tool + Parameters
   ↓
Core::preview_*() → Preview
   ↓
Risk Explainer + Artifact Renderer
   ↓
User confirmation
   ↓
Core::execute_*() → TxHash
   ↓
Response to user
```

## Stack Options

| Option | Pros | Cons |
|--------|------|------|
| **Rust (Rig)** | Type safety, same repo as core, fast | Young ecosystem, smaller community |
| **Python (FastAPI + LangChain)** | Mature ecosystem, huge community, easy prototyping | Another language, deployment overhead |

**Recommendation:** Start with Rust (Rig) for tight core integration. If Rig limits us — isolate LLM behind gRPC and swap implementation.
