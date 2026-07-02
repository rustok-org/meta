# Mobile UX Specification

> ⚠️ **Частично устарело — до разворота 2026-06-25 (custodial-эра).** Части про «ключ/подпись
> на сервере» неактуальны: продукт развёрнут на non-custodial device-signing. Текущее состояние —
> `PROJECT-STATUS.md`; незыблемое — `core/.claude/NORTH-STAR.md`. Держится как исторический контекст.

## Principle: Chat-First, Not Wallet-First

The app opens into a chat. The wallet is a tool the agent uses, not the main screen.

## Screens

### 1. Chat (Main Screen)

```
┌─────────────────────────────┐
│  9:41                       │
│                             │
│  [Rustok Agent]             │
│  Привет! Чем могу помочь?   │
│                             │
│  ───────────────────────    │
│                             │
│  [User]                     │
│  Отправь Алисе 0.5 ETH      │
│                             │
│  [Artifact: Send Preview]   │
│  ┌─────────────────────┐    │
│  │ To: 0x1234...5678   │    │
│  │ Amount: 0.5 ETH     │    │
│  │ Gas: $0.02          │    │
│  │ Risk: ✅ Safe       │    │
│  │                     │    │
│  │ [  Confirm  ]       │    │
│  └─────────────────────┘    │
│                             │
│  [Rustok Agent]             │
│  ✅ Отправлено. Tx: 0x...   │
│                             │
│  ───────────────────────    │
│                             │
│  [  🎤  | Type message... ] │
└─────────────────────────────┘
```

**Input methods:**
- Text
- Voice (hold microphone)
- Quick actions ("Send", "Swap", "Stake") above keyboard

### 2. Onboarding

Flow:
1. **Welcome** — logo + tagline
2. **PIN** — 6-digit, create
3. **Confirm PIN** — repeat
4. **Biometry** — FaceID / Fingerprint prompt
5. **Done** — "Ваш агент готов". Seed phrase available in Settings later.

No seed phrase on first launch. Embedded wallet by default.

### 3. Settings

- Security (biometry, PIN, seed phrase reveal)
- Networks (active chains)
- Address book
- Theme (light / dark / system)
- Language
- Agent personality (conservative / balanced / aggressive)

## Artifacts (Interactive Cards)

Artifacts render inside chat bubbles. They are not separate screens.

| Artifact | Content | Actions |
|----------|---------|---------|
| **Send Preview** | To, amount, gas, risk badge | Confirm / Edit / Cancel |
| **Swap Preview** | Route, price impact, slippage, min received | Confirm / Cancel |
| **Balance Card** | Total balance, per-chain breakdown, 7-day sparkline | Tap to expand |
| **Stake Card** | Protocol, APY, TVL, risk level | Confirm / Learn more |
| **Tx Receipt** | Tx hash, status, link to explorer | Share / Close |

## Navigation

Bottom tabs (minimal):
- **Chat** (active by default)
- **Activity** (history, not main)
- **Settings**

No "Wallet" tab. Balance shown via artifact in chat.

## Design Tokens

| Token | Light | Dark |
|-------|-------|------|
| Background | `#FFFFFF` | `#0A0A0F` |
| Surface | `#F5F5F7` | `#1C1C24` |
| Primary | `#8387C3` | `#8387C3` |
| Text primary | `#0A0A0F` | `#FFFFFF` |
| Text muted | `#6B6B7B` | `#8A8A9A` |
| Success | `#34C759` | `#34C759` |
| Warning | `#FF9500` | `#FF9500` |
| Danger | `#FF3B30` | `#FF453A` |
