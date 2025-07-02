>All of these where invalid findings but useful keep because I learnt a lot


### 1. **No Slippage Protection in Minting — Vulnerable to Front-Running**

**Impact:** High  
**Severity:** Medium-High

**Details:**  
The `mintDsc()` function allows users to mint DSC based on the latest Chainlink price without any slippage tolerance (e.g. `minAmountOut`, `minExpectedPrice`). This allows attackers or MEV bots to **front-run** honest users or manipulate price oracles briefly, minting at inflated prices before the feed updates.

**Example Exploit Path:**

1. Manipulate price to $10,000/ETH
    
2. Mint max DSC allowed
    
3. Price reverts to $2,000/ETH
    
4. Position becomes undercollateralized
    

**Recommendation:**  
Add a slippage check or allow the user to specify `minExpectedPrice`:


```js
equire(currentPrice >= userMinExpectedPrice, "Price slippage too high");`
```

---

###  2. **Oracle Price Can Be Manipulated — Enables Overminting**

**Impact:** High  
**Severity:** High

**Details:**  
The protocol relies solely on Chainlink’s `latestRoundData()` with no sanity checks, TWAP, or bounds. If the oracle price is temporarily manipulated (e.g., $2K → $10K per ETH), a user can mint **excessive DSC**, bypassing the intended collateralization constraints.

**This is a protocol-level flaw**: even if the oracle reverts shortly after, the position remains dangerously undercollateralized until liquidation kicks in, if at all.

**Recommendation:**

- Use a **TWAP oracle** or **median of recent rounds**
    
- Enforce **price bounds** (`minPrice`, `maxPrice`)
    
- Introduce a **circuit breaker** if price deviates by X% from previous value



**Recommendation:**  
Implement oracle defense mechanisms:

- Use **TWAPs** or **median-based pricing**
    
- Add **minPrice / maxPrice bounds**
    
- Introduce a **circuit breaker** if price changes too rapidly



### 3. **100% LTV on Oracle Value — Risk-Free Arbitrage During Volatility**

**Impact:** Medium-High  
**Severity:** Medium

**Details:**  
The protocol lets users mint DSC using **100% of the oracle-reported collateral value**. In volatile markets where Chainlink lags, this opens a profitable arbitrage:

- ETH trades at $1,800, but oracle still shows $2,000
    
- User deposits 1 ETH and mints $2,000 DSC
    
- Sells DSC on market and exits with a $200 gain
    

**This does not even require oracle manipulation**, just timing.

**Recommendation:**

- Apply a **minting fee, safety discount (e.g. 90% LTV)**
    
- Only allow minting below a **maxCollateralRatio**
    
- Combine with the oracle defense strategies above

This issue **erodes protocol stability** and can be repeatedly exploited during volatile markets. 




### 4. **Liquidation Griefing via Oracle Manipulation**

**Impact:** Medium–High  
**Severity:** High  
**Status:** Confirmed via test (`testLiquidationGriefingWithOracleManipulation`)

**Description:**  
A malicious actor can **temporarily manipulate the oracle price downward** (e.g., from $200 → $150) causing honest users’ health factors to drop **below the liquidation threshold**, **even if the collateral price returns to normal shortly after**.

This enables:

- **Griefing**: Causing unnecessary liquidations
    
- **Slashing**: Honest users lose collateral due to temporary, manipulated conditions
    
- **MEV Arbitrage**: A bot manipulates the oracle, liquidates victims, then restores the oracle—all within a single block


POC
```js
function testLiquidationGriefingWithOracleManipulation() public {

uint256 depositedCollateral = 1 ether;

uint256 mintAmount = 900e18;

  

// USER deposits 1 ETH at $2000 → $2000 collateral value

MockV3Aggregator(ethUsdPriceFeed).updateAnswer(2000e8);

ERC20Mock(weth).mint(address(this), depositedCollateral);

ERC20Mock(weth).approve(address(dsce), depositedCollateral);

dsce.depositCollateralAndMintDsc(address(weth), depositedCollateral, mintAmount);

  

// HEALTH FACTOR should be safe initially

uint256 initialHF = dsce.getHealthFactor(address(this));

console2.log("Initial HF:", initialHF);

assertGt(initialHF, 1e18);

  

// -------------- Attacker steps in -----------------

address attacker = address(1337);

  

// Attacker manipulates the oracle price (e.g., submits stale/low value)

vm.prank(attacker);

MockV3Aggregator(ethUsdPriceFeed).updateAnswer(1500e8); // Drop price → $1500 per ETH

  

// Now user looks undercollateralized (1 ETH * 0.5 threshold = $750 < $900 minted)

  

// Liquidator (attacker) prepares 1 DSC to liquidate

// dsc.mint(attacker, 1e18);

deal(address(dsc), attacker, 1e18);

vm.prank(attacker);

dsc.approve(address(dsce), 1e18);

  

// Liquidation should now succeed with just 1 DSC

vm.prank(attacker);

dsce.liquidate(address(weth), address(this), 1e18);

  

uint256 postHF = dsce.getHealthFactor(address(this));

console2.log("Post-Liquidation HF:", postHF);

}
```


