## `DexSwap Aggregator — Study Notes`

> The DexSwap Aggregator is a modular, smart routing system designed to facilitate optimized token swaps across multiple DEX aggregators **and** DeBank’s own custom route executor.

---

### 🔧 High-Level System Summary

- **Project**: DeBank – DexSwap Aggregator (Sherlock Contest)
    
- **Primary Goal**: Allow users to swap tokens via the best available route (split across DEXs or aggregators)
    
- **Architecture**: Modular with routers, adapters, and an executor
    

---

### 🏗️ Core Modules & Responsibilities

|Module|Purpose|
|---|---|
|`Router.sol`|Entry point. Manages swaps, fees, adapter selection|
|`Spender.sol`|Handles secure `delegatecall` to third-party adapters|
|`Adapters/`|Wrap 3rd-party DEX aggregators behind a unified interface|
|`Executor.sol`|Custom logic for splitting and routing trades across raw DEXs|
|`Library/`|Math, ERC20 logic, utilities|

---

### 📦 Supported Aggregators (Adapters)

These are third-party aggregators DexSwap wraps internally:

---

## 🧠 DexSwap Aggregator — Study Notes

📁 `05 - Audit Contest/sherlock/DeBank/study`

---

### 🧭 Overview

**DexSwap** is a high-level **DEX aggregator router** developed by **DeBank**, currently in a Sherlock audit contest. It allows efficient token swaps by routing through either:

- Third-party DEX aggregators (like 1inch, Paraswap), or
    
- DeBank's **own custom DEX aggregator**, which performs direct multi-DEX routing and trade splitting.
    

The design supports **flexible adapter registration**, **ETH/WETH abstraction**, and **custom fee collection logic**.

---

### 🏗️ Core Architecture

| Component                     | Description                                                                   |
| ----------------------------- | ----------------------------------------------------------------------------- |
| **Router (`DexSwap.sol`)**    | Main user entrypoint. Manages swaps, routing, fee logic, validation           |
| **Adapters (3rd-party)**      | Abstract interface to wrap various DEX aggregators                            |
| **Spender (`Spender.sol`)**   | Performs secure `delegatecall` to adapter contracts                           |
| **Executor (`Executor.sol`)** | Used by DeBank’s own aggregator logic. Handles route splitting, DEX selection |
| **Library/Utils**             | Math, ERC20/WETH helpers, decimals, slippage checks                           |

---

### 🔌 Third-Party Aggregators Supported

These are integrated via adapters in the `aggregatorRouter/adapter/` directory:

1. **1inch** (`OneinchAdapter.sol`)
    
2. **Uniswap Universal Router** (`UniAdapter.sol`)
    
3. **Paraswap** (`ParaswapAdapter.sol`)
    
4. **KyberSwap** (`KyberAdapter.sol`)
    
5. **Macha V2** (`MachaV2Adapter.sol`)
    
6. **Magpie** (`MagpieAdapter.sol`)
    

---

### 🧩 Internal Aggregator (DeBank Executor)

This is DeBank’s **custom aggregator** which uses:

- `Router.sol` + `Executor.sol` + direct `Adapter` structs
    
- Bypasses third-party APIs to split and execute trades across real DEXs
    

#### DEXs Supported via `swapType` inside `SimpleSwap`:

|DEX / Action|`swapType`|Notes|
|---|---|---|
|Uniswap V2|`1`|pool-based swap|
|Maker PSM|`2`|stablecoin swaps (USDC/DAI)|
|Curve V1|`3`|supports `exchange`, `exchange_underlying`|
|Curve V2|`4`|adds `GenericFactoryZap`, liquidity actions|
|Uniswap V3|`5`|pool + fee specific|
|Algebra V3|`9`|fork of UniV3|
|Uniswap V3 Fork|`10`|forked versions (same structure)|
|Balancer V1|`7`|uses pool address only|
|Balancer V2|`8`|uses `poolId` + address|
|Velodrome|`11`|router + tick-based logic|
|WETH Wrap/Unwrap|`11`|implicit behavior|
|Uniswap V4|`12`|hook-based with `UniversalRouter`|

---

### 🔁 Swap Execution Flow

> Applies to both adapter-based swaps and internal aggregator swaps


```
graph TD A[User calls swap()] --> B[Validate inputs + fee settings] B --> C[Wrap 
ETH to WETH (if needed)] C --> D[Split via Adapter(s) or Internal Aggregator] D --> E[Execute swaps across DEXs] E --> F[Check slippage protection] F --> G[Collect output fee (if any)] G --> H[Send toToken to user]
```

---

### 🧾 Structs to Know

#### `SwapParams`

Used in `DexSwap` (third-party aggregator swap)

```ts

struct SwapParams {
    string aggregatorId;
    address fromToken;
    uint256 fromTokenAmount;
    address toToken;
    uint256 minAmountOut;
    uint256 feeRate;
    bool feeOnFromToken;
    address feeReceiver;
    bytes data;
}

```
#### `MultiPath`, `SinglePath`, `Adapter`, `SimpleSwap`

Used in internal DeBank aggregator routing:
```js
struct MultiPath {
    uint256 percent;
    SinglePath[] paths;
}

struct SinglePath {
    address toToken;
    Adapter[] adapters;
}

struct Adapter {
    address payable adapter;
    uint256 percent;
    SimpleSwap[] swaps;
}

struct SimpleSwap {
    uint256 percent;
    uint256 swapType;
    bytes data;
}

```

---

### 🔐 Security Mechanisms

- ✅ `ReentrancyGuard` in critical contracts
    
- ✅ Fee logic applies to `fromToken` or `toToken`, defined per call
    
- ✅ ETH is always wrapped into WETH at entry, then unwrapped if needed
    
- ✅ Admin functions: `setAdapter`, `removeAdapter`, `pause()`, `unpause()`
    
- ✅ Input validation and slippage checks
    

---

### 🧠 Why This Protocol Is Interesting to Audit

- Complex adapter logic + external `delegatecall` (via `Spender`)
    
- Trade splitting and routing logic (can go wrong with decimals, slippage, rounding)
    
- Fee collection, precision, and misuse possibilities
    
- ETH/WETH wrapping and unwrap flows
    
- Opportunities for:
    
    - Front-running
        
    - Miscalculated routing
        
    - Swap path abuse (swapType spoofing?)
        
    - Fallback injection via low-level call/delegatecall
        

---

### 🧭 Next Steps (Your Own)

-  Open `Router.sol` → Trace `swap()` logic
    
-  Walk into `Executor.sol` → Inspect `_executeRoute` and swap dispatch logic
    
-  Follow `Spender.sol` and its `delegatecall` usage
    
-  Log any adapter function selector mismatches
    
-  Model fee logic and ETH ↔ WETH behavior
    

---

### 🏷️ Tags

#dex #aggregator #router #slippage #adapter-pattern #delegatecall #uniswap #curve #auditing

---

Let me know when you're ready to:

- Build your visual flow in Excalidraw
    
- Dive into the `Router.sol` swap call trace
    
- Get a checklist of high-risk audit targets (e.g., fee logic, input path integrity)