# 🛡️ Audit Log: PROJECT_NAME

> **Date Started:** 2025-06-28  
> **Auditor:** @0x00T1  
> **Target Commit:** `abc1234`  
> **Repo URL:** [GitHub](https://github.com/example/project)

---

## 🧠 High-Level Summary

- What does this protocol do?
- Core mechanism?
- Any external protocols it interacts with?
- Any economic levers / high-stakes flows?

---

## 🗂️ File Map & First Impressions

```text
src/
├─ Token.sol
├─ Vault.sol
├─ Strategy.sol
test/
├─ Vault.t.sol
├─ Strategy.t.sol
Token.sol — Basic ERC20 wrapper

Vault.sol — Main user interaction contract

Strategy.sol — External yield adapter
```


## 📌 Audit Progress Tracker

|File|Status|Notes|
|---|---|---|
|Token.sol|✅ Done|Standard ERC20, no surprises|
|Vault.sol|⏳ Reading|Complex share/accounting math|
|Strategy.sol|❌ Not yet|Suspect function: `withdrawFromAdapter()`|

## 🧩 Assumptions per File/Function

### Vault.sol

- `deposit()`  
  - Assumes `msg.value` matches amount?
  - Assumes reentrancy is not possible
  - Assumes token transfer always succeeds

- `withdraw()`  
  - Assumes no slippage
  - Assumes accurate share math

- `skim()`  
  - Assumes strategy holds surplus


## 🔍 Function/Call Tree


graph TD

```text
  deposit --> _mintShares
  deposit --> transferFrom
  withdraw --> _burnShares
  withdraw --> strategy.withdraw()

```
  




## 🎯 Areas of Interest (Attack Surface)

- [ ] External calls: `strategy.withdraw()`, `token.transfer()`
    
- [ ] Uses `delegatecall` or `call`?
    
- [ ] `unchecked` math blocks?
    
- [ ] Signature verification or Merkle proofs?
    
- [ ] Custom math logic?



## 🧠 Hypotheses & Mental Models

🔸 Guesses
Maybe `skim()` can be abused if someone inflates the strategy?

Suspect vault underflows if share math is off

Does `totalAssets()` always reflect reality?

🔸 Questions
Is `assetPrice()` ever manipulated?

Does it handle deflationary tokens?

Can someone bypass deposit limits?

## 🧪 Tests I Want to Try

 -  Try reentrancy in `withdraw()`

 -  Fuzz deposit + skim + withdraw to break accounting

 -  Manipulate token with `transferFrom` hook?

 -  Strategy returning incorrect balance?

## 🧨 Failed Ideas / Dead Ends

Track your failed assumptions — they matter.

❌ Thought skim() could be drained, but minAssets check prevents it

❌ Thought vault.withdraw() could underflow — nope, it uses SafeMath

❌ Signature replay — not possible, uses nonce++

📌 Confirmed Invariants
 totalAssets = totalShares * pricePerShare is respected ✅

 No external call before internal state update ✅

 All external calls are bounded ⏳

## 🔍 Tooling Outputs

Slither / MythX / Echidna notes

```text
- Slither: warns about public variable shadowing
- Echidna: will test `deposit`/`withdraw` round-trips
```


## 🧵 Threads / Code Patterns
### 🔁 Repeated Code:
```js
IERC20(token).safeApprove(spender, 0);
IERC20(token).safeApprove(spender, amount);
```


## 🧨 Dangerous Patterns:
`delegatecall()` without checks

onlyEOA modifiers

inline assembly {} blocks


## 📖 References & Crosslinks
[[01 - Daily Log/2025-06-26]] ← notes from that day

[[ERC4626 security model]]

[[Merkle tree proof pitfalls]]

## ✅ Final Summary / Outcome
Bugs found: 2 medium, 1 low

PoCs written: Yes, all verified in Foundry

PR submitted / Write-up sent

Overall Risk: 🔶 Medium
Review time: ~6 hours

Tags
#audit #shadow-audit #smartcontracts #PROJECT_NAME



---

## 🧠 Bonus Tips for Obsidian Flow

- Use `## 🔍 Function: XYZ` as **foldable sections** per function you're auditing
- Use `[[linked notes]]` to tie in research (e.g., "Two’s complement overflow")
- Store hypotheses in **callout blocks** for visual separation
- You can even create **`02 - audit templates/blank.md`** and clone for every project

---




