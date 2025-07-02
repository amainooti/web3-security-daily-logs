## Phase 1: Code Coverage 🧠

- Contracts:
  - [x] DSCEngine
  - [x] DSC
  - [x] OracleLib
- Function maps: ![[excalidraw/DSC-full-map]]


- Storage:
  - i_dsc: ✅ mint target
  - s_collateralDeposits: map of collateral → user
- Notes:
  - OracleLib is called via `delegatecall` → check context.
  - External user access via `mintDsc()`, `depositCollateral()`

---

## Phase 2: Actor Roles 👤

### Protocol
- Mints DSC based on 200% LTV
- Assumes oracle price is trustworthy

### User
- Can mint up to 50% LTV
- Can be liquidated under health factor < 1

### Liquidator
- Needs to have DSC *before* calling `liquidate`
- Gets collateral bonus

---

## Phase 3: Assumptions 🧠🔍

> Explicit protocol assumptions, to break later

- Oracle price is honest and fresh ❌
- Liquidators can always profit ❌
- User will not mint and exit right before price drop ❌
- Tokens have 18 decimals (→ check WBTC)

---

## Phase 4: Math Assignment & Roles 📐

| Actor | Function/Variable | Equation | Comment |
|-------|-------------------|----------|---------|
| Protocol | Health Factor | `(collateral in USD * threshold) / debt` | Decimal bug in WBTC breaks this |
| Liquidator | Bonus | `1.1 * debtPaid` | Not enough if DSC needed was hard to get |
| User | Mint | `collateral * 0.5` | No slippage guard → frontrunnable |

---

## Phase 5: Perspective Hopping 🔁

### From Liquidator
- ❌ Cannot recoup DSC due to minting restrictions
- ❌ Redeeming collateral breaks health factor

### From Oracle Attacker
- If I control price, I can:
  - inflate price → overmint
  - deflate after → leave bad debt
  - Frontrun updates

---

## Findings 📓

- [x] Liquidation Trap when 100%-110% collateralized
- [x] Token Decimal mismatch (WBTC mispriced)
- [x] No `minAmountOut` in `mintDsc()` → frontrunnable
- [x] OnlyOwner `burn()` bypassed via `burnFrom()`

---

## Hypotheses & Tests 🧪

- Hypothesis: Liquidator will lose value when forced to mint DSC → ✅ confirmed
- Hypothesis: Burnable bug exists in `DecentralizedStableCoin` → ✅ PoC working
- Hypothesis: Oracle delay allows frontrun minting → WIP test

---

## Reflection 🪞 (Talking to Self)

- Missed this at first because I wasn’t thinking like a liquidator.
- Next audit, simulate real-world steps:
  - Mint → wait → fall into liquidation zone → try liquidate
