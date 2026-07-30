# M0 Stablecoin Integration — AI Coding Skill

Use this skill when building stablecoin extensions on M0, integrating Wrapped M (`wM`), using SwapFacility, or bridging M tokens cross-chain.

---

## Architecture Overview

```
Your Custom Stablecoin (Extension)
        | SwapFacility.swap() / swapInM() / swapOutM()
    $M Token (M0 Foundation)
        | backed by
    US Treasury Collateral (SPVs)
```

1 Extension Token = 1 unit of value (always redeemable). Extensions inherit M0's stability, regulatory clarity, and yield.

---

## Extension Templates

### 1. Treasury Model — `MYieldToOne` (EVM, US/Regulated)

All yield flows to a single designated treasury wallet. Users hold a stable, non-rebasing ERC-20.

```solidity
// Check claimable yield
uint256 pending = myExtension.yield();

// Anyone can trigger yield claiming — mints new tokens to the configured recipient
myExtension.claimYield();

// Roles (AccessControl)
// YIELD_RECIPIENT_MANAGER_ROLE — update the yield recipient address
// FREEZE_MANAGER_ROLE          — freeze/unfreeze individual accounts (compliance)
```

**Source**: `MYieldToOne.sol`
**Good for**: Loyalty programs, B2B payments, protocol treasury management, US-regulated issuance.

---

### 2. Multi-Collateral Model — `JMIExtension` ("Just Mint It", Offshore)

Inherits all `MYieldToOne` features, plus the ability to accept multiple stablecoins (USDC, USDT, DAI) as collateral via SwapFacility.

```solidity
// Users wrap via SwapFacility — no direct mint function on JMIExtension
// SwapFacility calls wrap(asset, recipient, amount) internally
SwapFacility.swap(usdcAddress, jmiExtensionAddress, amount, recipient);

// Configure per-asset caps — requires ASSET_CAP_MANAGER_ROLE
myJMI.setAssetCap(USDC_ADDRESS, 10_000_000e6);   // 10M USDC cap (6 decimals)
myJMI.setAssetCap(DAI_ADDRESS,  5_000_000e18);   // 5M DAI cap (18 decimals)

// Rebalance: swap stablecoin collateral for $M (SwapFacility only)
myJMI.replaceAssetWithM(asset, recipient, amount);

// Yield claiming same as MYieldToOne
myJMI.claimYield();
```

**Critical**: Only set caps for stablecoins that maintain a reliable 1:1 peg. JMI assumes 1:1 peg for ALL accepted collateral — a depegged asset creates undercollateralization risk.

**Good for**: Offshore stablecoin launches prioritizing frictionless user onboarding.

---

### 3. NoYield (SVM — Solana/Fogo)

Treasury model for Solana. Admin claims all accumulated rewards via `claim_fees` instruction. Uses Token-2022 (or legacy Token Program).

```rust
// Only admin can call
claim_fees(recipient, amount);
```

**Source**: `github.com/m0-foundation/solana-m-extensions`
**Good for**: Protocol treasuries and ecosystem funds on Solana.

---

## Deploying an Extension

### Step 1: Inherit from the base contract

```solidity
import { MYieldToOne } from "./projects/yieldToOne/MYieldToOne.sol";

contract MyStablecoin is MYieldToOne {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor(address mToken_, address swapFacility_) MYieldToOne(mToken_, swapFacility_) {
        _disableInitializers();
    }

    function initialize(
        string memory name_,
        string memory symbol_,
        address yieldRecipient_,
        address admin_,
        address freezeManager_,
        address yieldRecipientManager_
    ) public initializer {
        MYieldToOne.initialize(name_, symbol_, yieldRecipient_, admin_, freezeManager_, yieldRecipientManager_);
    }
}
```

### Step 2: Deploy (use CREATE2 for deterministic address)

```solidity
// Using CREATE2 to pre-compute the address before deployment
bytes32 salt = keccak256(abi.encode("my-stablecoin-v1"));
address predicted = factory.computeAddress(salt, bytecodeHash);
address deployed = factory.deploy(salt, bytecode);
```

### Step 3: Get Earner Approval from TTG governance

For your extension to accrue yield, its deployed address must be approved as an M0 Earner:

1. Compute the final deployment address (via CREATE2 or testnet deploy)
2. Submit a `StandardGovernor` proposal to add the address to the `earners` list in `TTGRegistrar`
3. Wait for `$POWER` holders to vote and the proposal to pass (voting epoch is ~15 days)
4. Once approved, call `enableEarning()` on your deployed extension contract

**Without earner approval, yield does NOT accrue** — the contract holds $M but earns nothing.

### Step 4: Enable earning and initialize roles

```solidity
// Start yield accrual (call after earner approval is confirmed)
myExtension.enableEarning();

// Roles are set via initialize(); update them after deployment as needed
myExtension.grantRole(YIELD_RECIPIENT_MANAGER_ROLE, treasuryMultisig);
myExtension.grantRole(FREEZE_MANAGER_ROLE, complianceAddress);
```

---

## Core Extension Functions

`wrap()` and `unwrap()` are **called by SwapFacility only** — never call them directly from user-facing code.

```solidity
// Integrators always go through SwapFacility:
SwapFacility.swapInM(extensionAddress, amount, recipient);   // $M → extension
SwapFacility.swapOutM(extensionAddress, amount, recipient);  // extension → $M (permissioned)
SwapFacility.swap(extensionIn, extensionOut, amount, recipient); // extension ↔ extension

// Check claimable yield
uint256 pending = extension.yield();

// Claim yield (mints new tokens to the yield recipient — open to anyone)
extension.claimYield();

// Earning lifecycle (called by extension admin after earner approval)
extension.enableEarning();
extension.disableEarning();
```

---

## Wrapped M (`wM`) — `0x437cc33344a0B27A429f795ff6B469C72698B291`

Wrapped M is the reference implementation extension for standard DeFi compatibility. It is a non-rebasing ERC-20 that wraps `$M` and maintains yield-earning capability.

### Earning Modes

Accounts can be in **Non-Earning** or **Earning** mode:

```solidity
// Check if an account is earning
bool isEarning = wM.isEarning(account);

// Start earning (requires TTG-approved earner or admin delegation)
wM.startEarning();                    // for your own account
wM.startEarningFor(account);         // admin-delegated

// Stop earning
wM.stopEarning();

// Claim yield (yield accumulates separately from balance)
wM.claimFor(account);                // claim to account itself
wM.claimFor(account, recipient);     // claim to a custom recipient
```

### Balance Mechanics

- Non-earning balances are **static** — stored as raw token amounts, never change automatically.
- Earning balances are stored as **principal** — actual value = `principal × index`. Balance grows continuously.
- Yield accrues separately and must be claimed explicitly.

### Solvency Invariant

```
M Balance held by wM contract >= Total wM Supply + Total Accrued Yield
```

The wM contract always holds enough `$M` to back all circulating `wM` plus all unclaimed yield.

**Use `wM` when**: You need standard DeFi integration (AMM pools, lending protocols) and don't want to deploy your own extension.

---

## SwapFacility — Atomic Extension Swaps

`SwapFacility` enables 1:1 atomic swaps between any approved M0 Extensions with no slippage or fees.

```solidity
// Swap between two extensions
SwapFacility.swap(
    extensionIn,    // address: extension you're giving
    extensionOut,   // address: extension you want
    amount,         // uint256
    recipient       // address
);

// Swap $M → extension (open to everyone)
SwapFacility.swapInM(extensionOut, amount, recipient);

// Swap extension → $M (requires M_SWAPPER_ROLE or special permission)
SwapFacility.swapOutM(extensionIn, amount, recipient);

// Permit variants for gasless approvals
SwapFacility.swapWithPermit(extensionIn, extensionOut, amount, recipient, deadline, v, r, s);
```

### How it works

1. Validates both extensions are approved earners via `TTGRegistrar`
2. Pulls `extensionIn` tokens from caller
3. Calls `extensionIn.unwrap()` → receives `$M`
4. Calls `extensionOut.wrap()` → mints target tokens
5. Sends to `recipient`

All value relationships are 1:1 — no AMM pools needed between extensions.

---

## Cross-Chain Bridging

### M Portal (Wormhole) — EVM L2s

For Ethereum → Arbitrum/Base/Optimism/Linea and back:

```solidity
// Lock $M on source chain and receive bridged $M on destination
// Uses Wormhole NTT (Native Token Transfer)
// Bridged M earns yield via index propagation
MPortal.transferTokens(amount, destinationChainId, recipient);
```

### M Portal Lite (Hyperlane) — EVM L2s

Lighter-weight alternative using Hyperlane messaging. Same 1:1 peg; lower operational complexity for supported chains.

### Portal V2 — Unified Bridge

The latest bridge architecture with modular bridge adapters, supporting both token bridging and protocol metadata propagation (yield index, governance parameters). Enables M to earn yield on L2s by syncing the earner index from Ethereum mainnet.

### Solana

`$M` on Solana is bridged separately. The Solana SVM programs use Token-2022. Extensions on Solana use the `NoYield` or `ScaledUi` models depending on yield distribution requirements.

---

## Gaining Earner Approval — Checklist

1. **Determine deployment address** — use CREATE2 for a predictable address before deploying
2. **Submit TTG proposal** — add address to `earners` list in `TTGRegistrar` via `StandardGovernor.propose()`
3. **Wait for vote** — voting epochs are ~15 days; proposal needs simple majority of `$POWER` votes
4. **After approval** — call `enableEarning()` on your extension contract
5. **Verify** — `TTGRegistrar.listContains("earners", extensionAddress)` should return `true`

---

## Terminology

| Term | Meaning |
|---|---|
| Extension | A custom ERC-20 stablecoin built on M0 (wraps/unwraps `$M`) |
| `MYieldToOne` | EVM extension template: all yield → single treasury wallet |
| `JMIExtension` | EVM extension template: `MYieldToOne` + multi-collateral instant minting |
| NoYield | SVM extension template: admin-claimed yield on Solana |
| `wM` (Wrapped M) | Reference extension for DeFi compatibility (non-rebasing, claimable yield) |
| `SwapFacility` | Atomic 1:1 swap router between any approved M0 Extensions |
| `claimYield()` | Mints accumulated yield to the configured recipient |
| M Portal | Wormhole-based bridge for `$M` across EVM chains |
| M Portal Lite | Hyperlane-based bridge (lighter alternative) |
| Portal V2 | Unified bridge with modular adapters + index propagation |
| earner approval | TTG governance vote required for a contract to accrue `$M` yield |

---

## Common Pitfalls

- **Earner approval before yield** — yield does not accrue until governance approves and `enableEarning()` is called on the extension. Plan for the ~15-day voting lag.
- **JMI 1:1 assumption** — JMIExtension assumes all accepted collateral stablecoins maintain a perfect 1:1 USD peg. Only add assets you trust to hold their peg.
- **`claimYield()` mints new tokens** — it increases total supply. Factor this into your tokenomics and any cap logic.
- **wM yield is separate from balance** — `balanceOf()` on an earning wM account does NOT include unclaimed yield. Call `claimFor()` to realize it.
- **`swapOutM` is permissioned** — swapping extension → `$M` requires `M_SWAPPER_ROLE` or a per-extension permission. `swapInM` (`$M` → extension) is open.
- **Index must propagate cross-chain** — for bridged M to earn yield on L2, the earner index must be relayed from Ethereum mainnet. Portal V2 handles this automatically; older deployments may have lag.
