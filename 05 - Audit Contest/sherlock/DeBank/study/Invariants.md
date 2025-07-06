## Invariant Analysis Checklist (System-Wide)

Use this to formalize the **invariants** that should always hold — in audits, fuzzing, or formal verification.

---

### ✅ **Router Contract Invariants**

1. **FeeReceiver should never receive > feeRate% of input amount**  
    _Invariant_: `actualFee <= feeRate * totalAmountIn / 1e18`
    
2. **Swap cannot proceed while paused**  
    _Invariant_: `paused == true → revert all external swap calls`
    
3. **No adapter is used unless marked active**  
    _Invariant_: `usedAdapter ∈ getAdapters()`
    
4. **All `MultiPath.percent` values ≤ 1e18 and sum up to 1e18**  
    _Invariant_: `∑(multiPaths[].percent) == 1e18`
    
5. **All `SinglePath.percent` within a MultiPath sum to 1e18**  
    _Invariant_: `∑(singlePath[].percent) == 1e18`
    
6. **No unauthorized address can call `pause()` or `unpause()`**  
    _Invariant_: `caller ∈ admins[]`
    

---

### ✅ **Executor Contract Invariants**

1. **fromToken allowance must be sufficient for each adapter**  
    _Invariant_: `IERC20(fromToken).allowance(executor, router) >= amountIn`
    
2. **No call to `forceApprove` with ETH as fromToken**  
    _Invariant_: `fromToken != ETH → ok, else → handled separately`
    
3. **Swap fails if `adapter.executeSimpleSwap` returns false or reverts**  
    _Invariant_: `swap outcome is revert → no tokens are lost/stuck`
    

---

### ✅ **Adapter Contracts (V2/V3)**

1. **Only supported swapTypes are executed**  
    _Invariant_: `swapType ∈ {1, 5, ...}`
    
2. **Correct decoding of payload matches swapType**  
    _Invariant_: `swapType == X → abi.decode(payload, TypeForX)`
    
3. **No reentrancy into Router from Adapter**  
    _Invariant_: _Requires_ `nonReentrant` or no external call back into router
    
4. **Output token ≠ 0 if the pool is valid**  
    _Invariant_: `amountOut > 0 → tokenOut received`
    

---

### ✅ **Admin Module Invariants**

1. **Router cannot be unpaused if ≥2 admins are paused**  
    _Invariant_: `∑(adminPauseStates) ≥ 2 → router.paused == true`
    
2. **Only current admins can update the admin list**  
    _Invariant_: `caller ∈ admins[] → router.updateAdmins() allowed`
    
3. **Cannot assign zero address or duplicate admin**  
    _Invariant_: `newAdmin ≠ 0x0 && !admins.contains(newAdmin)`
    

---

### ✅ **Token & Approval Safety**

1. **No infinite allowance unless explicitly intended**  
    _Invariant_: `allowance ≤ totalSupply || isForceApproveSafe == true`
    
2. **All tokens transferred in → used or refunded**  
    _Invariant_: `final balance ≥ initial balance - expected delta`
    

---

### ✅ **Misc Invariants**

- **Swap paths resolve to known token pairs**  
    _Invariant_: For each hop: `(tokenIn, tokenOut, fee) ∈ knownPools`
    
- **No WETH sent to contract expecting ETH**  
    _Invariant_: `weth → unwrap → send ETH only where needed`