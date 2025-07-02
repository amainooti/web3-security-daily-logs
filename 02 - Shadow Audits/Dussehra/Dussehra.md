# 🏹 Dussehra Protocol – Audit Report

**Date Audited:** June 10, 2025  
**Auditor:** Amaino Oti  
**Protocol Name:** Dussehra Protocol  
**Project Type:** Cultural NFT Lottery Game

---

## 🧠 High-Level Overview

The Dussehra Protocol is a gamified cultural NFT experience inspired by Indian mythology. Users pay an entrance fee to enter a pool via `peopleWhoLikeRam()`, hoping to be selected as **Ram**. A pseudo-random selection determines the **Ram**, who can then challenge another participant to “kill Ravana.” If victorious, they mint a unique **RamNFT** and claim honor. The core mechanics blend pseudo-randomness, participation fees, and NFT minting.

The project is spread across three contracts:

- `RamNFT`: ERC-721 contract for minting the Ram NFT
    
- `ChoosingRam`: Logic for selecting Ram from entrants
    
- `MainDussehraGame`: Handles user entry, selection, challenge, and victory flow
    

---

## ✅ Positive Design Patterns

- ✅ Simple modular separation between game logic and NFT minting
    
- ✅ Pseudo-random selection flow (albeit flawed) using `block.prevrandao`
    
- ✅ Emitted events for key actions (minting, selection, challenge outcomes)
    

---

## 🚨 Key Findings

### 🟥 High Severity

#### 1. 🚫 Missing Access Control on `mintRamNFT()`

solidity

CopyEdit

`function mintRamNFT(address receiver, uint256 id) external {     _mint(receiver, id); }`

- **Impact**: Anyone can mint unlimited NFTs without restrictions.
    
- **Consequence**: NFT inflation, trust loss, and dilution of game mechanics.
    
- **Fix**: Add `onlyAuthorized` modifier — ideally only `MainDussehraGame` or similar.
    

---

#### 2. 🧠 Uncapped NFT Supply

- No mechanism to limit the total number of `RamNFT`s minted.
    
- **Impact**: Infinite NFTs dilute scarcity and undermine narrative/utility.
    
- **Fix**: Add `MAX_SUPPLY` and `require(_tokenId < MAX_SUPPLY, ...)`.
    

---

#### 3. 🔁 Missing Replay Protection in `killRavana()`

solidity

CopyEdit

`function killRavana() public {     isRavanKilled = true; }`

- **Impact**: Repeated calls allow re-killing Ravana and re-triggering state updates.
    
- **Fix**: Add `require(!isRavanKilled, "Ravana already defeated")`.
    

---

#### 4. 🧍 Self-Challenge Allowed — Same Challenger and Participant

- No check prevents user from setting themselves as both `challenger` and `participant`.
    
- **Impact**: Guaranteed victory with no opposition; undermines core fairness.
    
- **Fix**: Add `require(challenger != participant, "Cannot challenge self")`.
    

---

#### 5. 🎲 Biased Randomness — Challenger Has Higher Win Probability

solidity

CopyEdit

`uint256 rand = uint256(block.prevrandao) % 2; // 0 = participant wins, 1 = challenger wins`

- **Impact**: With only two outcomes and fixed mapping, one role always has 50%+ chance.
    
- **Fix**: Replace with verifiable randomness (e.g., Chainlink VRF) or a fairer distribution.
    

---

### 🟧 Medium Severity

#### 6. 🔄 Missing State Update: `isSelected = true`

- After `selectRam()`, selected address is not marked as such.
    
- **Impact**: User can be selected multiple times unfairly.
    
- **Fix**: Add `s_isSelected[user] = true` post-selection.
    

---

#### 7. 🔓 No `ReentrancyGuard` on Sensitive External Functions

- Functions like `killRavana()` and ETH-handling functions lack `nonReentrant` modifier.
    
- **Impact**: Opens reentrancy attack surface, especially if future changes add transfers.
    
- **Fix**: Add OpenZeppelin’s `ReentrancyGuard` and mark sensitive functions.
    

---

### 🟨 Low Severity

#### 8. 💸 ETH Mishandling — No Fee Validation

solidity

CopyEdit

`function peopleWhoLikeRam() public payable {     // Missing check }`

- **Impact**: Users can enter with wrong ETH amounts or send excess ETH without refund.
    
- **Fix**: Add `require(msg.value == entranceFee, "Wrong ETH sent")`.
    

---

## 🛠 Recommendations

- ✅ Add `onlyGameContract` to `mintRamNFT()`
    
- ✅ Cap minting with `MAX_SUPPLY`
    
- ✅ Add `require(!isRavanKilled)` in `killRavana()`
    
- ✅ Prevent self-challenges (`challenger != participant`)
    
- ✅ Rethink randomness source; consider Chainlink VRF or entropy from block variables
    
- ✅ Patch missing `isSelected` state update
    
- ✅ Use `ReentrancyGuard` on external+stateful functions
    
- ✅ Add strict `msg.value` validation for fair entry