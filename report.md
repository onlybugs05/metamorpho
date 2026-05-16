# Bug Report — MetaMorpho (`/home/runner/work/metamorpho/metamorpho/src/MetaMorpho.sol`)

## 1) Potential withdrawal DoS when a market is insolvent

### Summary
`MetaMorpho._withdrawable` computes available liquidity with a raw subtraction:

```solidity
totalSupplyAssets - totalBorrowAssets
```

If `totalBorrowAssets > totalSupplyAssets` for a market (bad debt / insolvency edge state), this underflows and reverts in Solidity 0.8+, which can break withdrawal-related flows.

### Affected code
- File: `/home/runner/work/metamorpho/metamorpho/src/MetaMorpho.sol`
- Function: `_withdrawable`
- Relevant lines:

```solidity
uint256 availableLiquidity = UtilsLib.min(
    totalSupplyAssets - totalBorrowAssets, ERC20(marketParams.loanToken).balanceOf(address(MORPHO))
);
```

### Impact
- Withdrawal logic that depends on `_withdrawable` (directly or via simulation paths) can revert unexpectedly.
- Users may be unable to withdraw in stressed market conditions, causing denial of service.

### Likelihood
- Low to Medium (depends on whether insolvency/bad-debt states can be reached in integrated markets).

### Severity assessment
- **Medium** (DoS impact on withdrawals in an adverse but plausible state).

### Recommended remediation
Use safe floor subtraction instead of raw subtraction, e.g.:
- `totalSupplyAssets.zeroFloorSub(totalBorrowAssets)`, or
- conditional branch returning `0` when `totalBorrowAssets >= totalSupplyAssets`.

This preserves behavior while preventing arithmetic revert in insolvency states.

---

## 2) `maxDeposit` / `maxMint` can overestimate capacity due to duplicate `supplyQueue` entries

### Summary
`setSupplyQueue` allows duplicate markets, and `_maxDeposit` iterates and sums each queue entry independently.  
If a market appears multiple times, remaining cap can be counted more than once, causing `maxDeposit` / `maxMint` to overestimate actual depositability.

### Affected code
- File: `/home/runner/work/metamorpho/metamorpho/src/MetaMorpho.sol`
- Functions:
  - `setSupplyQueue` (duplicates allowed)
  - `_maxDeposit`
  - `maxDeposit`, `maxMint`

### Impact
- Integrators relying on ERC-4626 limits may attempt deposits/mints that revert.
- User-facing systems may display incorrect safe limits and degrade UX/reliability.

### Likelihood
- Medium (configuration-dependent; occurs when allocator submits duplicate IDs).

### Severity assessment
- **Low** (primarily integration/operational risk; documented warning exists).

### Recommended remediation
Either:
1. Enforce uniqueness in `setSupplyQueue`, or
2. Deduplicate IDs while computing `_maxDeposit`.

This aligns `maxDeposit`/`maxMint` with actual executable capacity and ERC-4626 integrator expectations.

