TODO: Talk to myself about router contract






Spender contract can only be called by `dexSwap`
It uses a `delagatecall` Because adapters need access to **Spender’s storage context** (e.g., token balances).
the `_delegatecall` function is a low level call that assumes the adapter errors are bubbled up 

Currently observing the aggregator looked at the `swap` and `_delegatecall`
then I went right into the dexSwap


| Feature                  | **DexSwap AggregatorRouter (swap via Adapters)**        | **In-House Aggregator (Router + Executor + MultiPath)**   |
| ------------------------ | ------------------------------------------------------- | --------------------------------------------------------- |
| Contract File            | `DexSwap.sol` in `aggregatorRouter/`                    | `Router.sol` in `router/`                                 |
| Swap Function            | `DexSwap::swap(SwapParams)`                             | `Router::swap(fromToken, ..., MultiPath[])`               |
| Path Param               | `SwapParams.data` (raw `bytes`)                         | `MultiPath[] paths` (explicit, structured)                |
| Aggregator Used          | External (1inch, Paraswap, Kyber, etc) via **Adapters** | Internal execution via **Executor.sol** (custom router)   |
| Delegate Call Target     | `Spender` ➝ `Adapter`                                   | Calls `executor.executeMegaSwap(...)`                     |
| Who does the swap logic? | **Adapter** contract                                    | **Executor** contract                                     |
| Uses Adapter registry    | ✅ Yes (`aggregators[adapterId]`)                        | ❌ No                                                      |
| Flexibility              | Swaps through external protocols                        | Custom logic to split/schedule trades internally          |
| Trade path logic format  | Raw `bytes data` handled inside adapter                 | Rich `MultiPath → SinglePath → Adapter → SimpleSwap` tree |
| Handles ETH              | Via `UniversalERC20`, special ETH address               | Same                                                      |




## 🔄 Adapter1 — Cross-Chain Differences

Both `mainnet/Adapter1.sol` and `xdai/Adapter1.sol` implement the `IAdapter` interface and dispatch swaps to multiple DEX-specific executors.

---

### 🧩 Mainnet Adapter1
**DEXs Supported:**
- 1. Uniswap V2
- 2. Maker PSM
- 3. Curve V1
- 4. Curve V2
- 5. Uniswap V3
- 6. Velodrome
- 7. Balancer V1
- 8. Balancer V2
- 9. Algebra V3
- 10. Uniswap V3 Fork
- 11. WETH Swap
- 12. Uniswap V4

**Constructor**: Takes `_dai`, `_weth`, `_permit2` addresses  
**Lines of Inheritance**: 13 DEX Executors

---

### 🌐 xDai Adapter1
**DEXs Supported:**
- 1. Uniswap V2
- 3. Curve V1
- 4. Curve V2
- 5. Uniswap V3
- 8. Balancer V2
- 9. Algebra V3
- 10. Uniswap V3 Fork
- 11. WETH Swap
- 12. Uniswap V4

**Constructor**: Takes only `_weth`, `_permit2`  
**Lines of Inheritance**: 10 DEX Executors



|`mainnet/Adapter1.sol`|`xdai/Adapter1.sol`|
|---|---|---|
|**swapType 2**|✅ Maker PSM (`swapMakerPsm`)|❌ _Not supported_|
|**swapType 6**|✅ Velodrome|❌ _Not supported_|
|**swapType 7**|✅ Balancer V1|❌ _Not supported_|
|**swapType 8**|✅ Balancer V2|✅ Balancer V2|
|**swapType 12**|✅ Uniswap V4|✅ Uniswap V4|
|**Constructor**|Needs `_dai`, `_weth`, `_permit2`|Only `_weth` and `_permit2`|
|**Inherits**|13 Executors|10 Executors|

---

### 🔐 Security Perspective
- Protocol uses scoped adapter contracts per chain
- Avoids registering unsupported DEXs on L2s
- Keeps contract lean and tailored to liquidity available

---

### 📎 Used in:
- Aggregator-style routing via `DexSwap → Spender → Adapter`
- Selector-based dynamic execution
- Supports raw bytes calldata (fuzz surface)




| Role               | Key Abilities                                             | Controlled Contracts            |
| ------------------ | --------------------------------------------------------- | ------------------------------- |
| **Owner**          | Owns `Executor`, can call `executeMegaSwap()`             | `Executor.sol`                  |
| **Admin (3x)**     | `pause()`, `unpause()`, `setAdapter()`, `removeAdapter()` | `DexSwap`, `Admin.sol`          |
| **User**           | Initiates swaps, provides tokens, receives output         | `Router`, `DexSwap`             |
| **Spender**        | Middleware, restricted to Router                          | `Spender.sol`                   |
| **Adapters**       | Execute actual swaps on DEXs                              | `UniAdapter`, `KyberAdapter`... |
| **Permit2**        | Offchain-like permission approval system                  | Used in adapters                |
| **Executor Owner** | Controls `Executor`, likely a router/admin                | `Executor.sol`                  |


Key interaction

        [User]
          |
          |------------------------.
          |                        |
     [Router]                 [DexSwap]
          |                        |
      [Executor]              [Spender]
          |                        |
  delegatecall                  delegatecall
  to Adapter                    to Adapter
          |                        |
          '-----------.------------'
                      |
                   [Adapter]
       (Contains logic for interacting with DEXs)
     ┌────────────────────────────────────────────┐
     │ UniswapV2, UniswapV3, CurveV1, CurveV2,    │
     │ Balancer, Velodrome, AlgebraV3, MakerPSM   │
     └────────────────────────────────────────────┘
                      |
                      V
                 [External DEXs]




## 🧱 CONTRACTS & ROLES


[User / Frontend]
   └──> Triggers swaps via:
           ├─[Router.swap()]       // for MultiPath route
           └─[DexSwap.swap()]      // for single-DEX route


### 🔷 MULTIPATH ROUTE FLOW

[Router]
    │   - Validates inputs
    │   - Optionally takes fee (on fromToken)
    └──> [Executor]
             │   - Calculates balance deltas
             │   - For each MultiPath:
             │       └──> For each SinglePath:
             │               └──> For each Adapter:
             │                       └─delegatecall─> [Adapter1]
             │                              ├──> swapUniswapV2()
             │                              ├──> swapCurveV1()
             │                              ├──> ...
             │
             └─ Final balance diff returned to Router
    │
    └──> Router optionally takes fee (on toToken)
         and sends funds back to user


### 🔷 DEXSWAP ROUTE FLOW
[DexSwap]
    │   - Validates inputs via _validateSwapParams()
    │   - Charges fee on fromToken if set
    │
    └──> Transfers token to [Spender]
             │
             └──> [Spender.swap()]
                        │
                        └─delegatecall─> [Adapter1]  <── Set dynamically via setAdapter()
                               ├──> swapUniswapV2()
                               ├──> swapCurveV2()
                               ├──> ...
             │
             └──> Resulting toToken is sent back to DexSwap
    │
    └──> Charges fee (on toToken, if needed)
         and sends toToken to user


## 🧩 OTHER COMPONENTS
[Admin]
    ├── Can pause/unpause protocol with multi-admin consensus
    ├── Can replace their own admin address (if not paused)
    └── Is inherited by both Router and DexSwap

[Adapter1 (xDai / mainnet)]
    ├── Holds core DEX interaction logic
    ├── Has function: executeSimpleSwap(...)
    ├── Dispatches via swapType enum:
    │     ├── 1  → UniswapV2
    │     ├── 2  → MakerPSM
    │     ├── 3  → CurveV1
    │     └── ...
    └── Each swapType maps to a corresponding:
         [DEXExecutor.sol] via inheritance
