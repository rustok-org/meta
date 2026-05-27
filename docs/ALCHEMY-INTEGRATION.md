# Alchemy Integration Plan

## Why Alchemy (not self-hosted revm)

| Factor | Alchemy | Self-hosted revm |
|--------|---------|------------------|
| Accuracy | Exact on-chain state | Requires full state sync |
| Latency | ~200ms | ~0ms (local) |
| Maintenance | Zero | High (state updates, patches) |
| Bundle size | No impact | +10-15 MB |
| Offline | No | Yes |

**Decision:** Alchemy as primary. Standard `eth_call` as fallback. No revm.

## Endpoints Used

### 1. `alchemy_simulateAssetChanges`

**Purpose:** txguard simulation (replaces revm).  
**What it shows:** ETH balance change, token changes, approvals, NFTs, gas used, revert status.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "alchemy_simulateAssetChanges",
  "params": [{
    "from": "0x...",
    "to": "0x...",
    "value": "0x...",
    "data": "0x..."
  }]
}
```

**Response:**
```json
{
  "changes": [
    { "assetType": "ERC20", "contractAddress": "0x...", "from": "0x...", "to": "0x...", "amount": "-1000000000000000000" }
  ],
  "gasUsed": "21000",
  "reverted": false
}
```

### 2. `alchemy_getTokenBalances`

**Purpose:** Show all token balances for an address.  
**Replaces:** Manual multicall for each token.

### 3. `alchemy_getAssetTransfers`

**Purpose:** Transaction history (activity screen).  
**Replaces:** Manual block scanning.

## Fallback Strategy

```
Primary:   Alchemy simulateAssetChanges
Fallback:  eth_call + custom decoding (for simple transfers)
Emergency: Static analysis (calldata parsing without simulation)
```

## Rate Limiting

- Free tier: 100 compute units/second
- Growth tier: 200 CU/s
- SimulateAssetChanges: ~200 CU per call

**Mitigation:** Cache preview results for 30 seconds. Batch balance queries.

## Implementation

```rust
pub struct AlchemyProvider {
    client: reqwest::Client,
    api_key: String,
    base_url: String, // https://eth-mainnet.g.alchemy.com/v2/
}

impl AlchemyProvider {
    pub async fn simulate(&self, tx: &TransactionRequest) -> Result<Simulation, Error>;
    pub async fn token_balances(&self, address: Address) -> Result<Vec<TokenBalance>, Error>;
    pub async fn transfers(&self, address: Address) -> Result<Vec<Transfer>, Error>;
}
```

## Security Note

Alchemy sees calldata. For privacy-sensitive operations (e.g., private transfers), user can switch to direct RPC.
