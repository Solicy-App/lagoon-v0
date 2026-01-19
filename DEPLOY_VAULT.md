# Deploying BUNN Vault on Base Sepolia

## Prerequisites

1. **Foundry installed** - [Install Foundry](https://book.getfoundry.sh/getting-started/installation)
2. **Base Sepolia ETH** - Get from [Base Sepolia Faucet](https://www.alchemy.com/faucets/base-sepolia)
3. **Basescan API Key** - Get from [Basescan](https://basescan.org/myapikey)

## Setup

1. Copy environment file and configure:

```bash
cp .env.example .env
```

2. Edit `.env` with your values:

```env
PRIVATE_KEY=0xYOUR_PRIVATE_KEY
BASE_SEPOLIA_RPC_URL=https://sepolia.base.org
BASESCAN_API_KEY=YOUR_BASESCAN_API_KEY
CHAIN_ID=84532
```

## Configure Vault Parameters

Edit `script/DeployVault1_USDC.s.sol`:

```solidity
// Safe wallet address (curator - receives deposited funds and has admin control)
address constant SAFE = 0xYOUR_SAFE_ADDRESS;

// Vault token details
string constant VAULT_NAME = "Vault Shares USDC";
string constant VAULT_SYMBOL = "vsUSDC";
```

### Role Configuration

All roles are assigned to the Safe wallet for full control:

| Role | Description |
|------|-------------|
| `safe` | Receives deposited USDC |
| `admin` | Can update vault settings |
| `whitelistManager` | Manages deposit whitelist |
| `valuationManager` | Updates vault valuation |
| `feeReceiver` | Receives management/performance fees |

## Deploy

```bash
forge script script/DeployVault1_USDC.s.sol:DeployVault1_USDC \
  --rpc-url https://sepolia.base.org \
  --broadcast
```

## Verify Contracts

After deployment, verify the implementation contract:

```bash
forge verify-contract <IMPLEMENTATION_ADDRESS> \
  src/v0.5.0/Vault.sol:Vault \
  --chain base-sepolia \
  --etherscan-api-key $BASESCAN_API_KEY \
  --constructor-args 0x0000000000000000000000000000000000000000000000000000000000000000 \
  --watch
```

## Deployment Output

The script outputs:

```
CONTRACT ADDRESSES:
-------------------
FeeRegistry:       0x...
Implementation:    0x...
Factory/Beacon:    0x...
BUNN Vault:        0x...  <-- Use this in your frontend
```

## Update Frontend

After deployment, update the vault address in `BUNN-Vault/packages/shared/src/constants.ts`:

```typescript
export const BASE_SEPOLIA_ADDRESSES = {
  USDC: "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
  LAGOON_VAULT: "0xYOUR_NEW_VAULT_ADDRESS",
  TREASURY: "0xYOUR_TREASURY_ADDRESS",
  SAFE: "0xYOUR_SAFE_ADDRESS",
};
```

## Enable SyncDeposit Mode (REQUIRED after deployment)

After deploying a new vault, syncDeposit is NOT enabled by default. The vault starts in async mode where users must call `requestDeposit()` and wait for settlement.

To enable syncDeposit (instant deposits), the **Safe wallet must execute three transactions**:

### Option A: Using Safe App UI (Recommended)

1. Go to your Safe wallet at [app.safe.global](https://app.safe.global)
2. Connect to Base Sepolia network
3. Go to "Apps" → "Transaction Builder"
4. Execute these three transactions in order:

**Transaction 1: Set totalAssetsLifespan**
- To Address: `0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2` (Vault)
- ABI: `[{"inputs":[{"internalType":"uint128","name":"lifespan","type":"uint128"}],"name":"updateTotalAssetsLifespan","outputs":[],"stateMutability":"nonpayable","type":"function"}]`
- Function: `updateTotalAssetsLifespan`
- lifespan: `3153600000` (100 years in seconds)

**Transaction 2: Propose new total assets**
- To Address: `0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2` (Vault)
- ABI: `[{"inputs":[{"internalType":"uint256","name":"_newTotalAssets","type":"uint256"}],"name":"updateNewTotalAssets","outputs":[],"stateMutability":"nonpayable","type":"function"}]`
- Function: `updateNewTotalAssets`
- _newTotalAssets: `0` (for fresh vault)

**Transaction 3: Settle and initialize**
- To Address: `0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2` (Vault)
- ABI: `[{"inputs":[{"internalType":"uint256","name":"_newTotalAssets","type":"uint256"}],"name":"settleDeposit","outputs":[],"stateMutability":"nonpayable","type":"function"}]`
- Function: `settleDeposit`
- _newTotalAssets: `0` (must match Transaction 2)

### Option B: Using cast CLI

If your Safe is an EOA (single signer) or you have direct access:

```bash
# 1. Set lifespan (100 years)
cast send 0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2 \
  "updateTotalAssetsLifespan(uint128)" 3153600000 \
  --rpc-url https://sepolia.base.org \
  --private-key $SAFE_PRIVATE_KEY

# 2. Propose new total assets
cast send 0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2 \
  "updateNewTotalAssets(uint256)" 0 \
  --rpc-url https://sepolia.base.org \
  --private-key $SAFE_PRIVATE_KEY

# 3. Settle deposit to initialize totalAssetsExpiration
cast send 0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2 \
  "settleDeposit(uint256)" 0 \
  --rpc-url https://sepolia.base.org \
  --private-key $SAFE_PRIVATE_KEY
```

### Verify SyncDeposit is Enabled

```bash
cast call 0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2 \
  "isTotalAssetsValid()(bool)" \
  --rpc-url https://sepolia.base.org
```

Should return `true`. If it returns `false`, syncDeposit won't work.

## Current Deployed Contracts (Base Sepolia)

| Contract | Address |
|----------|---------|
| BUNN Vault | `0x6A81075A7cA4a1fd8c1ABF2098C1ab72330743B2` |
| Implementation | `0x06c3AFD4aAaF69D09682831771AD35a2896549f9` |
| FeeRegistry | `0xB4dA79496eA62F1dF95CccFB75D25b21e0F0DeBA` |
| Factory/Beacon | `0x28F024A4877701ff3a362344512C7Af08fa17E65` |
| Safe (Admin) | `0xFc3c513cE3aD237939085ede9097a3D2141eBAF9` |
| USDC | `0x036CbD53842c5426634e7929541eC2318f3dCF7e` |

## Troubleshooting

### "forge: command not found"
Add Foundry to PATH or use full path:
```bash
~/.foundry/bin/forge script ...
```

### Verification fails
- Ensure correct constructor args
- For proxy contracts, verify the implementation instead
- Check Basescan API key is valid

### Deployment fails
- Ensure enough Base Sepolia ETH for gas
- Check private key is correct in `.env`
