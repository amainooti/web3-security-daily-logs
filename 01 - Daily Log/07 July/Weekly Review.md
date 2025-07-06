## 🧾 Weekly Review: **June 30 – July 6, 2025**

### 📅 **Sunday (June 30)** – _Closing the Month Strong_

- ✅ Finished reviewing the **Foundry Stablecoin Audit Report**
    
- 📌 Reflected on missed findings, internalized techniques
    
- 🚀 Built deeper understanding of **lending & borrowing mechanics**
    
- 👁️ Learned to spot key bugs like:
    
    - Incorrect collateral accounting
        
    - Improper liquidation thresholds
        
    - Precision loss in stable conversions
        
    - Oracle manipulation vectors
        
    - Mint/burn logic flaws
        

---

### 📅 **Monday (July 1)** – _Core Research Mode_

- 🧠 Began studying **EVM internals**, **assembly**, and **gas optimization**
    
- Topics touched:
    
    - Stack & memory model
        
    - Storage layout
        
    - `mstore`, `sload`, `calldataload`, etc.
        
    - Common Yul patterns for gas savings
        
- 🔧 Realized the value of learning how compilers translate high-level code to low-level behavior.
    

---

### 📅 **Tuesday (July 2)** – _Pivot: Contest Activation_

- 🛑 Paused assembly/EVM learning
    
- ⚔️ Joined a new contest (Day 1)
    
    - Read through most of the system
        
    - Identified entry points, key execution paths
        
    - Verified existing test coverage
        
    - Started building offensive test cases
        

---

### 📅 **Wednesday–Saturday (July 3–6)** – _Active Contest Work_

- 🧪 Deep-dived into:
    
    - Router logic (multi-hop swap paths, fee manipulation)
        
    - Adapter/Executor relationships
        
    - State transitions during `pause/unpause`
        
    - FeeReceiver abuse vectors
        
    - Pool configuration validations (e.g., invalid UniswapV3 fee tiers)
        
- 🛠️ Built and fixed test scaffolding (forked tests, fuzz tests, negative cases)
    
- 📈 Slowly expanding list of known issues + acceptable risks
    

---

### 📅 **Sunday (July 6)** – _Reset + Reflect_

- 🧘 Took a step back from active contesting
    
- ✅ Documented all known bugs, risks, and expected behavior
    
- 🧠 Cleaned up mental model of:
    
    - Executor/Adapter dispatching
        
    - Swap routing invariants
        
    - Admin pause/upgrade coordination
        
- 📝 Performed a **weekly review**
    
- ✍️ Planning to resume EVM/Assembly course _after_ contest ends
    

---

## 📚 **What You Learned This Week**

- 🔍 **Audit report digestion**: Reverse engineering reviewer mindset
    
- 🛠️ **EVM mechanics**: Stack, memory, calldata, gas model
    
- 🧬 **Gas optimization**: Compiler output awareness, redundant operation detection
    
- 💣 **Exploit surfacing**: Execution-path analysis in routers/adapters
    
- 🔐 **Admin architecture**: Multi-sig-like pausing, privilege escalation checks
    
- 🧭 **Time/energy management**: You intentionally paced today (Sunday) to avoid burnout — **excellent**.


