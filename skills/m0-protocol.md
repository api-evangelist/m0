# M0 Protocol — AI Coding Skill

Use this skill when writing code that interacts with the M0 Protocol core contracts: minting/burning `$M`, collateral management, TTG governance, or earner yield.

---

## What Is M0?

M0 is an open, federated protocol for building custom stablecoins backed by eligible real-world collateral (U.S. Treasury Bills held in Special Purpose Vehicles). It provides the secure, collateralized foundation that stablecoin extensions build on top of.

The core protocol lives on Ethereum mainnet. Extensions can deploy to any EVM chain or Solana.

---

## Core Contracts (Ethereum Mainnet)

| Contract | Address |
|---|---|
| `$M` Token (`MToken`) | `0x866A2BF4E572CbcF37D5071A7a58503Bfb36be1b` |
| `MinterGateway` | `0xf7a27FE3579C420575a09D279f78CA6a3E5e75D0` |
| `TTGRegistrar` (Registrar) | `0x119FbeeDD4F4f4298Fb59B720d5654442b81ae2c` |
| `StandardGovernor` | `0xd0C5194740cDb03ee44c45Aa36E3C2e40aC95e39` |
| Wrapped `$M` (`wM`) | `0x437cc33344a0B27A429f795ff6B469C72698B291` |

GitHub repos:
- Protocol: `github.com/m0-foundation/protocol`
- TTG (governance): `github.com/m0-foundation/ttg`
- Wrapped M: `github.com/m0-foundation/wrapped-m-token`

---

## Key Roles

### Minters
Permissioned institutions that create `$M` by posting eligible offchain collateral. They pay interest on minted supply (minter rate), which funds earner yield and protocol revenue.

### Validators
Independent trusted entities that verify Minter collateral offchain and provide cryptographic signatures for `updateCollateral()`. They can freeze Minters or cancel suspicious mint proposals.

### Earners
Addresses (users, smart contracts, or M0 Extensions) approved by TTG governance to accrue yield on their `$M` balances. Must call `startEarning()` on the `MToken` contract. Yield compounds continuously at the earner rate.

### Power Token (`$POWER`) Holders
Vote on day-to-day protocol parameters in `StandardGovernor` (minter/validator/earner approvals, rates, ratios). Inflationary — 10% per voting epoch for participants who vote on all proposals.

### Zero Token (`$ZERO`) Holders
Meta-governance: can redeploy governance itself via `ZeroGovernor`. Receive pro-rata protocol revenue from the `DistributionVault`.

---

## Minting `$M` — Step by Step

Minters follow a propose → delay → execute pattern:

```solidity
// 1. Ensure collateral is up to date (within UPDATE_COLLATERAL_INTERVAL, ~30h)
MinterGateway.updateCollateral(
    collateral,         // uint256 collateral value
    retrievalIds,       // uint256[] pending retrieval IDs to resolve
    metadataHash,       // bytes32 metadata hash
    validators,         // address[] validator addresses (ascending order)
    timestamps,         // uint256[] validator timestamps
    signatures          // bytes[] EIP-712 validator signatures
);

// 2. Propose a mint
uint48 mintId = MinterGateway.proposeMint(amount, destination);

// 3. Wait MINT_DELAY (governance-set) — validators can cancel during this window

// 4. Execute before MINT_TTL expires
MinterGateway.mintM(mintId);
```

### Key constraints
- Collateral must be updated within `UPDATE_COLLATERAL_INTERVAL` (currently ~30h)
- `MINT_RATIO` governs max `$M` outstanding vs. verified collateral
- Missed updates or undercollateralization trigger penalties added to debt
- Penalties: `PENALTY_RATE * principalOfActiveOwedM * missedIntervals`

---

## Burning `$M`

Any `$M` holder can burn to reduce a Minter's debt:

```solidity
MinterGateway.burnM(minterAddress, amount);
```

For deactivated Minters, this is the informal liquidation mechanism — their `inactiveOwedM` is fixed (no longer accruing interest).

---

## Earning Yield

```solidity
// Requires the address to be approved as an earner in TTGRegistrar
MToken.startEarning();      // start accruing yield
MToken.stopEarning();       // stop (keeps accrued yield)

// Check balance (includes accrued yield for earning accounts)
uint256 bal = MToken.balanceOf(account);
```

The earner rate is always ≤ minter rate (capped at 98% of the safe rate). Yield compounds continuously — no claim transactions needed, balances grow automatically.

---

## TTG Governance

Two-Token Governance (TTG) controls all protocol parameters stored in `TTGRegistrar`.

Key parameters governed:
- `mint_ratio` — max $M per unit of collateral
- `base_minter_rate` — interest rate for Minters
- `max_earner_rate` — cap on earner yield
- `update_collateral_interval` — required collateral update frequency
- `update_collateral_threshold` — minimum validator signatures required
- Approved minters, validators, and earners lists

Reading a parameter:
```solidity
bytes32 value = TTGRegistrar.get(paramKey);
bool isApproved = TTGRegistrar.listContains("minters", minterAddress);
```

`StandardGovernor` proposals require simple majority (yes > no votes) from `$POWER` holders.
`ZeroGovernor` proposals require a minimum threshold of `$ZERO` votes for fundamental changes.

---

## Validator Signature Format

Collateral updates require multi-sig from approved validators:

```
- EIP-712 typed data signatures, validator-specific
- Validator addresses must be provided in ascending order (prevents duplicates)
- Each signature timestamp must be newer than the last from that validator for that minter
- Timestamp must not be in the future; must not be zero
- Minimum threshold of valid signatures required (governance-set)
```

---

## Interest & Index Mechanics

Both Minter debt and Earner balances use **continuous indexing** (stored as principal × index):

- `MinterGateway.updateIndex()` — synchronizes both the minter debt index and MToken earner index
- Called automatically on `updateCollateral()`, `mintM()`, `burnM()`, `deactivateMinter()`
- `excessOwedM` (minter interest minus earner yield) is minted to `DistributionVault` on each index update

---

## Terminology

| Term | Meaning |
|---|---|
| `$M` | The native yield-bearing stablecoin |
| `activeOwedM` | Minter's current debt including accrued interest |
| `inactiveOwedM` | Deactivated minter's fixed debt (no longer accruing) |
| `earningPrincipal` | Stored principal for earning accounts (actual balance = principal × index) |
| TTG | Two-Token Governance — the governance system (`$POWER` + `$ZERO`) |
| SPV | Special Purpose Vehicle — the offchain entity holding collateral |
| `MINT_RATIO` | Max $M a Minter can have outstanding relative to verified collateral |
| `MINT_DELAY` | Required wait between `proposeMint()` and `mintM()` |
| `MINT_TTL` | Expiry window for a mint proposal |

---

## Common Pitfalls

- **Earner approval is required** before yield accrues — calling `startEarning()` on an unapproved address reverts.
- **Collateral expiry**: if `UPDATE_COLLATERAL_INTERVAL` is missed, onchain collateral value drops to zero for all calculations.
- **Validator order**: addresses must be ascending in `updateCollateral()` — unsorted inputs revert.
- **Mint proposals expire**: a `mintId` becomes invalid after `MINT_TTL` seconds; a new proposal is required.
- **Deactivation is permanent**: deactivated Minters cannot be reactivated.
- **$M is NOT a user-facing yield product** — it is the collateral base layer. End-user yield comes from Extensions (e.g., Wrapped M).
