
**Date Audited:** June 7, 2025  
**Auditor:** Amaino Oti  
**Protocol Name:** Weather Witness  
**Project Type:** Dynamic ERC-721 NFT Collection powered by Chainlink Functions & Automation  
**Codebase:** Custom smart contract integrating Chainlink Functions, Chainlink Keepers, IPFS metadata URIs, and dynamic NFT weather updates. see [[Weather NFT]]

---

## 🧠 High-Level Overview

Weather Witness is a weather-reactive NFT project that allows users to mint NFTs based on real-time weather data fetched via Chainlink Functions. It also supports automated updates via Chainlink Automation (Keepers) to change the state and appearance of NFTs over time depending on weather changes. Each NFT is tied to a specific location (`pincode`, `isoCode`) and can optionally register an upkeep to automate metadata updates.

---

## ✅ Positive Design Patterns

- ✅ **Chainlink Functions Integration** to fetch real-time weather data in a decentralized way.
    
- ✅ **Automated Upkeep Registration** tied to NFTs, allowing periodic updates.
    
- ✅ **Metadata fully on-chain** using base64-encoded JSON in `tokenURI`.
    
- ✅ **Separate storage layer** (`WeatherNftStore`) for modularity and upgradeability.
    
- ✅ **Use of enums** (`Weather`) to represent weather states cleanly.
    
- ✅ **Gas-efficient storage of URI mappings** (`Weather => string` and reverse).
    

---

## 🚨 Findings

### 🟥 High Severity

1. **Unrestricted Mint Request (DoS Vector)**
    
    - `requestMintWeatherNFT()` lacks access control. Any address can initiate minting, potentially spamming the system.
        
    - This is especially dangerous due to the unbounded number of mint requests.
        
2. **Replay Attack via `fulfillMintRequest`**
    
    - No check exists to prevent a reused `requestId`.
        
    - An attacker can observe a valid requestId and call `fulfillMintRequest` multiple times, minting unlimited NFTs.
        
3. **Incorrect Mint Recipient (Frontrunning Risk)**
    
    - `fulfillMintRequest` mints to `msg.sender` instead of the original requester.
        
    - Attackers can front-run legitimate users and steal their NFT.
        
4. **Multiple Tokens per `requestId`**
    
    - Without nullifying the requestId post-mint, calling `fulfillMintRequest()` multiple times results in many NFTs per request.
        
5. **Unbounded Token Inflation**
    
    - No cap exists on the number of NFTs that can be minted. Combined with a lack of `requestId` tracking and open minting, this can lead to runaway inflation.
        

---

### 🟧 Medium Severity

6. **Missing Withdrawal Function**
    
    - Contract collects ETH via minting but lacks a `withdraw()` method.
        
    - Funds are permanently locked unless the contract is upgraded.
        
7. **No Maximum Cap on Mint Price Escalation**
    
    - `s_currentMintPrice` increases with every mint via `s_stepIncreasePerMint`, but there's no upper bound.
        
    - Could lead to runaway mint costs, pricing out users.
        
    - **Recommendation:** Add a constant `MAX_MINT_PRICE` and assert against it.
        

---

### 🟨 Low Severity

8. **No Validation on Weather Input**
    
    - The returned weather integer from Chainlink Functions is blindly cast into the `Weather` enum.
        
    - Could lead to invalid enum values if unexpected data is returned.
        
9. **Typo in JSON metadata**
    
    - `"Weathear NFT"` instead of `"Weather NFT"` in tokenURI generation.
        
10. **Inefficient Storage of Weather Mappings**
    

- Reverse mapping `string => Weather` is unused in the main contract and adds unnecessary storage overhead.
    

---

## 🛠 Recommendations

- ✅ Track requestId → bool status (used/unfulfilled) to prevent replay
    
- ✅ Add access control or mint caps per user to prevent spam
    
- ✅ Always mint to `s_funcReqIdToUserMintReq[reqId].user`, not `msg.sender`
    
- ✅ Add an upper bound (`MAX_MINT_PRICE`) to mint price escalation
    
- ✅ Add `withdraw()` function for owner or DAO treasury
    
- ✅ Nullify requestId data after `fulfillMintRequest` is called
    
- ✅ Add `require(tokenCounter < MAX_SUPPLY, "...")` if scarcity is intended
    
- ✅ Use try/catch when decoding responses from Chainlink Functions
    

---