# 🧪 Shadow Audit – Alchemix Strategy (2024-12)

- **NSLOC**: ~253
- **Goal**: Find bugs in strategy logic leveraging transmuter + alETH depeg
- **Source**: Contest repo from Dec 2024
- **Date Started**: 2025-06-26

---

## 🧠 High-Level Summary

Alchemix strategy using Yearn V3 strategy template to earn yield on alETH by:
- Depositing into Alchemix Transmuter
- Keeper triggers claiming of alETH→WETH
- Swaps WETH → alETH to profit from depeg

---

## 📌 External Functions

| Function | Purpose | Access Control |
|----------|---------|----------------|
|          |         |                |

_(To be filled)_

---

## 🧱 Observations & Hooks

- [ ] How is slippage handled during WETH → alETH swap?
- [ ] What if alETH never repegs? Are funds stuck?
- [ ] Keeper incentives?
- [ ] What if transmuter fails / halts?

---

## 🔐 Attack Surface

_TBD after initial mapping._

---

## 🧪 Bugs / Findings

_(Leave blank for now, update as you go)_
