# 📆 June 2025 Review

## ✅ Wins

- Found critical bugs in the Cyfrin Boss Bridge challenge
    
- Completed 9/10 Damn Vulnerable DeFi challenges
    
- Shadow audited multiple First Flights and identified real bugs
    
- Audited my **first live First Flight** — found a **High severity bug**
    
- Ranked **Top 100** in that First Flight contest
    
- Built my own evolving auditing workflow with Excalidraw, GPT, Obsidian, and Foundry
    
- Reviewed a 1K+ NSLOC protocol comfortably for the first time (Silo Finance)
    
- Fell in love with Obsidian and made it central to my workflow


---

## 🧠 Concepts Mastered

- ERC4626’s `totalAssets()` and how misinterpreting it can misrepresent vault TVL
    
- Slippage protection mechanics in bridges and how to validate their absence
    
- The danger of mixing `Ownable` + `ERC20Burnable` (e.g., `burnFrom()` bypassing `onlyOwner`)
    
- Pattern recognition for missing state updates and access controls
    
- Mental modeling for pseudo-randomness risks and how to test for biased randomness
    

---

## 🐛 Top Bugs I Found (and how)

- **Cyfrin Boss Bridge**
    
    - Missing slippage protection → Found by thinking “what if the bridge doesn’t revert?”
        
    - Missing access control → Checked function visibility manually
        
    - Incorrect TVL calculation → Spotted by checking ERC4626 integrations and missing invested tokens
        
- **Dussehra Protocol**
    
    - Missing access control & event emissions
        
    - Missing state update
        
    - Pseudo-random generation (biased randomness)
        
    - No checks to prevent self-play (i.e., playing as both Ram & participant)
        
    - Minor typos
        
- **Dating dApp**
    
    - State update bug (`likes[msg.sender] = true;` had no guard)
        
    - Missing access control again
        
- **First live First Flight**
    
    - Found a valid **High**
        
    - Other findings got **invalidated**, but still learned a lot from each one
        

---

## 💡 Failed Exploits That Taught Me Something

- **Foundry Stablecoin Audit**  
    I only caught 1 out of ~16 reported bugs. Here’s why:
    
    1. Gaps in my DeFi understanding
        
    2. Didn’t fully **perspective-switch** (“As a liquidator, can I...?”)
        
    3. Didn’t test **math edge cases** or bounds properly
        
    4. Overlooked how inherited external functions like `burnFrom()` can bypass restrictions
        

---

## 🔁 Systems & Habits That Worked

- Using **Excalidraw** to map key flows and state changes
    
- Using GPT to untangle tricky logic and test helpers
    
- Logging **every audit** in Obsidian with structured daily logs
    
- Doing **shadow audits** of past contests to train myself under realistic conditions
    
- Manual code review always comes first — tools only support my thinking
    

---

## 🧭 Energy + Flow Observations

- **Best hours:** 6 PM – 2 AM
    
- **What triggered flow state:**
    
    - Working with music + blackout mode
        
    - Mapping contracts visually in Excalidraw
        
    - Reading high-quality audit reports + bug bounty submissions before starting my review
        
- **What killed flow:**
    
    - Reading too many Medium/Low findings back to back (mental fatigue)
        
    - Not switching gears after 3 hours of non-stop auditing
        
    - Letting the fear of "missing bugs" make me skim code instead of understanding it