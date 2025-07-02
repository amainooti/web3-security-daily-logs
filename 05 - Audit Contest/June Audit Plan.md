## 🧠 June Audit Plan (June 8 - 30)

### 🎯 Goal

Prepare for real Audit contest submissions by sharpening auditing skills through structured shadow audits, deep-dive research, and attacker-mindset CTFs.

### ✅ Weekly Themes

**Each week blends audits, research, CTFs, and bug consolidation.**

---

### 📅 Week 1: June 8 - 14 — "Audit Flow Mastery"

- **Primary Audit**: Shadow audit a recent medium-to-large C4 project (e.g., Zircuit, Kinto, or similar)
    
- **Secondary**: Skim report after 3 days, document missed bugs
    
- **CTF**: Complete 1–2 security challenges from Paradigm/CTF repo
    
- **Research**:
    
    - Review `SafeERC20`, ERC-20 spec, edge cases
        
    - Read 2–3 known audit reports involving ERC-20 + MetaTx
        
- **Logging**:
    
    - Daily vault notes
        
    - Missed bug vault entries
        

---

### 📅 Week 2: June 15 - 21 — "Roles, Auth, MetaTx, Fees"

- **Primary Audit**: Protocol using multiple roles (Controller, Minter, Admin, Owner, etc.)
    
- **Focus**: Role overreach, authorization gaps, trust assumptions
    
- **CTF**: Capture Ethernaut / DamnVulnDeFi level with auth or proxy logic
    
- **Research**:
    
    - EIP-712, domain separator logic
        
    - `ecrecover`, `isValidSignatureNow`, replay prevention
        
- **Bug Library**: Start `MetaTx`, `Role Escalation`, `Allowance Bugs` categories
    

---

### 📅 Week 3: June 22 - 28 — "Governance, Upgrades & Vaults"

- **Primary Audit**: Governance-enabled protocol (e.g., Lido/Compound fork or custom gov logic)
    
- **Secondary**: Review OpenZeppelin Governor.sol deeper
    
- **CTF**: Realistic voting delay or token-based voting attacks
    
- **Research**:
    
    - Upgradeable patterns: UUPS vs Transparent proxy
        
    - Common vault bugs (slippage, permissions, logic errors)
        
- **Write**: Obsidian entry for upgrade-related checklist & voting sanity rules
    

---

### 📅 Week 4: June 29 - 30 — "Compression & Review"

- **Review**:
    
    - All bugs found and missed — consolidate Obsidian vault
        
    - Update Bug Library with tags and quick identifiers
        
- **Write**:
    
    - 3–5 new audit checklists
        
    - Reflection on which bugs you missed and why
        

---

### 🧰 Tools To Use

- **Obsidian**: Daily log, audit notes, CTF wins, bug library
    
- **Foundry**: For PoCs, tests, and replaying old exploits
    
- **Solodit**: Daily searches around bug themes you're learning
    

### 🗂️ Obsidian Structure Recap

- `00 - Inventory`

- `01 - Daily Log`
    
- `02 - Shadow Audits`
    
- `03 - CTFs`
    
- `04 - Bug Library`

- `05 - Audit Contest

- `06 - Research`
    
- `07 - Bounty Bounty`
    

---

Stay hungry. Start each week with an attack vector in mind. Always log what you missed — that’s where the gold is.