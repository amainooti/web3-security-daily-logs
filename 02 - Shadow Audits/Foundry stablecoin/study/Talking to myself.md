
## Overview
###  **Readme.md** 

This is Lesson 12 of the [Ultimate Foundry 27-hour Solidity Course](https://www.youtube.com/watch?v=umepbfKp5rI).

This project is meant to be a stablecoin where users can deposit WETH and WBTC in exchange for a token that will be pegged to the USD. The system is meant to be such that someone could fork this codebase, swap out WETH & WBTC for any basket of assets they like, and the code would work the same.


### Core In-variants

1. It's designed to have a *1-token-to-1-dollar* peg
2. Exogenous Collateral
3. Algoritmically Stable



## 💬 2025-06-28

It is similar to Dai basically If it had no governance, no fees and backed by only *WETH and WBTC*
see  [[DSC-engine.sol]]  

- DSC is the name of this stable coin



####  Thinking and understanding

`LIQUIDATION_THRESHOLD` the LT was set to 50 and this 50 represents the percentage limit that your collateral should not fall below;

`threshold = (collateral * LIQUIDATION_THRESHOLD) / LIQUIDATION_PRECISION >= debt`

Given a collateral of $2000 deposited and I wanted to borrow $700
collateral = 2000
debt = 700

`threshold = (2000 * 50/100)  = 1000`

`threshold = 1000`

`threshold > debt `

this is valid debt

Watch what happens when its not a valid debt 

`debt = 1300`

`threshold = 1000`

`debt > threshold = liquidation`



q? There's something called `MIN_HEALTH_FACTOR` and it sets to `1e18` I wonder what that does?

so research says so far that based on the absence of floating point numbers in solidity's its using a fixed point number arithmetic's to try and compensate for this.

1e18 = 1.0 
2e18 = 2.0


```js
/*

* @param tokenCollateralAddress The address of the token to deposit as collateral

* @param amountCollateral The amount of collateral to deposit

* @param amountDscToMint The amount of decentralized stablecoin to mint

* @notice this function will deposit your collateral and mint DSC in one transaction

*/

function depositCollateralAndMintDsc(

address tokenCollateralAddress,

uint256 amountCollateral,

uint256 amountDscToMint

) external {

depositCollateral(tokenCollateralAddress, amountCollateral);

mintDsc(amountDscToMint);

}
```


When they say Amount DSC to mint does that mean a user can deposit say $2000
and still specify How much he wants to mint Ok lets take an example

I deposit `WETH` so ;
@param 1 weth
@param 2 1 ether ($2000)
@param 3 12 ($12)


so debt is $12 and collateral is $2000 this is what they mean?  
I think I get it we shall test how much I understand this later on though



```js
// q? Does this actually check allowed tokens?
modifier isAllowedToken(address token) {

if (s_priceFeeds[token] == address(0)) { //q? <@ Isn't this a zero address check?

revert DSCEngine__NotAllowedToken();

}

_;

}
```



Apparently its checking if the token address is in the storage variable 
`mapping(address token => address priceFeed) private s_priceFeeds; // tokenToPriceFeed`

So its a zero address check but not a normal one if the price feed does not return it returns a zero address rather and then this check can revert




Trying to understand why that `MIN_HEALTH_FACTOR` is used 

```js
function _calculateHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)

internal

pure

returns (uint256)

{

if (totalDscMinted == 0) return type(uint256).max;

uint256 collateralAdjustedForThreshold = (collateralValueInUsd * LIQUIDATION_THRESHOLD) / LIQUIDATION_PRECISION;

return (collateralAdjustedForThreshold * 1e18) / totalDscMinted;

}
```


Looking at that `_calculateHealthFactor` 

`
- `collateralValueInUsd = 2000e18` (you deposited 1 ETH @ $2,000)
    
- `totalDscMinted = 1000e18` (you minted $1,000 worth of DSC)
    
- `LIQUIDATION_THRESHOLD = 50`
    
- `LIQUIDATION_PRECISION = 100`

`
`collateralAdjustedForThreshold = (2000e18 * 50) / 100
                              `= 1000e18`


`healthFactor = (1000e18 * 1e18) / 1000e18 
			`= 1000e36/1000e18`	
             `= 1e18 (or 1.0 in decimal)`



So if it falls below 1.0 the user health factor is too low

```js
function _revertIfHealthFactorIsBroken(address user) internal view {

uint256 userHealthFactor = _healthFactor(user);

if (userHealthFactor < MIN_HEALTH_FACTOR) {

revert DSCEngine__BreaksHealthFactor(userHealthFactor);

}

}
```



```js
function _calculateHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)

internal

pure

returns (uint256)

{

if (totalDscMinted == 0) return type(uint256).max;

uint256 collateralAdjustedForThreshold = (collateralValueInUsd * LIQUIDATION_THRESHOLD) / LIQUIDATION_PRECISION;

return (collateralAdjustedForThreshold * 1e18) / totalDscMinted;

}
```

I was curious as to why if the user has not minted any DSC tokens infinite number should be returned then I researched a bit and found out this is a clever short circuiting that says " hey if you've not minted then you have no debts but we cant return zero for you cause thats going to mess up our accounting instead hey your health check if infinite 

Butttt I was thinking why not set the max health check rather than be cheeky ? like have a literal variable that is a `MAX_HEALTH_FACTOR` maybe just for this function 





```js
function getTokenAmountFromUsd(address token, uint256 usdAmountInWei) public view returns (uint256) {

// price of ETH (token)

// $/ETH ETH ??

// $2000 / ETH. $1000 = 0.5 ETH

AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);

(, int256 price,,,) = priceFeed.staleCheckLatestRoundData();

// ($10e18 * 1e18) / ($2000e8 * 1e10)

return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);

}
```

So Chainlink returns 8 decimals and an additional precision is required to make it `value * 1e8 *1e10`  


```price = 2000 * 1e8 = 200000000000  // ETH price in 8 decimals

// Convert to 18-decimal precision by multiplying by 1e10
priceAdjusted = price * 1e10 = 200000000000 * 1e10 = 2000e18

// Now do:
return (usdAmountInWei * PRECISION) / (price * ADDITIONAL_FEED_PRECISION)
= (1000e18 * 1e18) / (2000e8 * 1e10)
= (1000e36) / (2000e18)
= 0.5e18

```




Currently looking at this `liquidate` function appaz the protocol allows an external actor to pay debts owned by Bad users or users that fall short of the LT 

step by step walk through of what this function does

- **Bad user’s position**:
    
    - Collateral = 0.07 WETH = $140
        
    - Debt = 100 DSC = $100
        
    - Health factor ≈ 0.7 → undercollateralized
        
- Liquidator sends `debtToCover = 100 DSC`



### 1️⃣ **Health Factor Check**

```js
uint256 startingUserHealthFactor = _healthFactor(user);

if (startingUserHealthFactor >= MIN_HEALTH_FACTOR) {
    revert DSCEngine__HealthFactorOk();
}

```

- If the borrower’s health factor is **≥ 1e18 (i.e. 1.0)** → revert
    
- Liquidations are only allowed **if undercollateralized**
    

✅ Good.

### 2️⃣ **Figure out how much collateral to give for that debt**

```js
uint256 tokenAmountFromDebtCovered = getTokenAmountFromUsd(collateral, debtToCover);

```
If:

- `collateral = WETH`
    
- `ETH price = $2000`
    
- `debtToCover = 100 DSC = $100`

```js
tokenAmountFromDebtCovered = (100e18 * 1e18) / (2000e8 * 1e10)
                           = 0.05e18 = 0.05 ETH

```

### 3️⃣ **Apply the liquidation bonus (10%)**

```js
bonusCollateral = (tokenAmountFromDebtCovered * LIQUIDATION_BONUS) / LIQUIDATION_PRECISION;
totalCollateralToRedeem = tokenAmountFromDebtCovered + bonusCollateral;

```

```js
bonus = 0.05 * 10 / 100 = 0.005 ETH
totalCollateral = 0.05 + 0.005 = 0.055 ETH

```

✅ The liquidator gets 0.055 ETH for paying off $100 DSC.

### 4️⃣ **Transfer collateral and burn DSC**

```js
_redeemCollateral(user, msg.sender, collateral, totalCollateralToRedeem);
_burnDsc(debtToCover, user, msg.sender);

```

- The liquidator burns their own `100 DSC`
    
- Collateral is pulled from the borrower and given to the liquidator


## 5️⃣ **Post-liquidation checks**

```js
uint256 endingUserHealthFactor = _healthFactor(user);
if (endingUserHealthFactor <= startingUserHealthFactor) {
    revert DSCEngine__HealthFactorNotImproved();
}
```

📌 If liquidation **didn’t improve** the user’s health factor → revert.

✅ This protects against unnecessary or abusive liquidation.


### 6️⃣ **Make sure the liquidator didn’t wreck their own health factor**

```js
_revertIfHealthFactorIsBroken(msg.sender);

```
If the liquidator had a leveraged DSC position too, this ensures they don’t break safety by becoming undercollateralized post-liquidation.



I'm just thinking these kind of maths are susceptible to rounding errors, precision loss etc Have no concrete hypothesis yet can't see any visible bug 



The last things I saw before calling it a day in this protocol were the redeem functions 


```js
function _redeemCollateral(address from, address to, address tokenCollateralAddress, uint256 amountCollateral)

private

{

s_collateralDeposited[from][tokenCollateralAddress] -= amountCollateral;

emit CollateralRedeemed(from, to, tokenCollateralAddress, amountCollateral);

bool success = IERC20(tokenCollateralAddress).transfer(to, amountCollateral);

if (!success) {

revert DSCEngine__TransferFailed();

}

}
```

Basic flow

Let’s say a user wants to withdraw:

- 0.5 ETH ($1000)
    
- Burns 1000 DSC
    

This function:

- Burns 1000 DSC
    
- Reduces `s_collateralDeposited[user][WETH]` by 0.5 ETH
    
- Transfers 0.5 ETH to user
    
- Re-checks health factor (must remain ≥ 1e18)





## 💬 2025-06-29


Thinking about the liquidate function and I just thought what if users that have a broken health factor are tracked in a datastructure or counted can be useful for the protocol 


Done with recon for the DSC-engine.sol [[DSC-engine.sol]]

```js

function getUsdValue(address token, uint256 amount) public view returns (uint256) {

AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);

(, int256 price,,,) = priceFeed.staleCheckLatestRoundData();

// 1 ETH = $1000

// The returned value from CL will be 1000 * 1e8

return ((uint256(price) * ADDITIONAL_FEED_PRECISION) * amount) / PRECISION;

}
```

```js

function getTokenAmountFromUsd(address token, uint256 usdAmountInWei) public view returns (uint256) {

// price of ETH (token)

// $/ETH ETH ??

// $2000 / ETH. $1000 = 0.5 ETH

AggregatorV3Interface priceFeed = AggregatorV3Interface(s_priceFeeds[token]);

(, int256 price,,,) = priceFeed.staleCheckLatestRoundData();

// ($10e18 * 1e18) / ($2000e8 * 1e10)

return (usdAmountInWei * PRECISION) / (uint256(price) * ADDITIONAL_FEED_PRECISION);

}
```




I'm looking at this and yes it does check stale price but what else can we do here ? is this bullet proof

While asleep I thought of few things

1. this protocol might be susceptible to flash loan looking at the deposit flow theres no snapshot no time weighted price although chainlink aggregator was used

Key things we have to attempt to do

1. Find accounting edge cases to break

2. See how we can exploit this with flashloan 

3. front running, back running or sandwitch attacks





So I came back to attempt attacking the protocol went through the test suite and decided to come up with attacks


```js
function testCannotDustGriefLiquidate() public {

uint256 someCollateralAmount = 1e16;

  

// Setup user

vm.startPrank(user);

ERC20Mock(weth).approve(address(dsce), amountCollateral);

dsce.depositCollateralAndMintDsc(weth, amountCollateral, amountToMint);

vm.stopPrank();

  

// Trigger undercollateralization

MockV3Aggregator(ethUsdPriceFeed).updateAnswer(18e8);

  

// Setup liquidator with 1 DSC

ERC20Mock(weth).mint(liquidator, someCollateralAmount);

vm.startPrank(liquidator);

ERC20Mock(weth).approve(address(dsce), someCollateralAmount);

dsce.depositCollateralAndMintDsc(weth, someCollateralAmount, 1); // Mint 1 DSC

  

dsc.approve(address(dsce), 1);

  

// Expect revert if dust liquidation not allowed

//vm.expectRevert(); // Optional: check revert reason

dsce.liquidate(weth, user, 1);

vm.stopPrank();

}
```


Tried griefing it but it reverted on account of unimproved health see below;

```text
    └─ ← [Revert] DSCEngine__HealthFactorNotImproved()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 7.11ms (1.17ms CPU time)

Ran 1 test suite in 12.27ms (7.11ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/unit/DSCEngineTest.t.sol:DSCEngineTest
[FAIL: DSCEngine__HealthFactorNotImproved()] testCannotDustGriefLiquidate() (gas: 614765)
```



I'll check accounting errors and edge cases last those are never fun to do 




So far so good I found some design bugs but I haven't found any highs no accounting logic flaws or anything like that 


## findings

1. No slippage in the minting hence can be frontrunned 
2. Is susceptible to Oracle manipulation


```js
function depositCollateralAndMintDsc(

address tokenCollateralAddress,

uint256 amountCollateral,

uint256 amountDscToMint

) external {

depositCollateral(tokenCollateralAddress, amountCollateral);

mintDsc(amountDscToMint);

}
```

#### Attack Steps:

1. **Attacker manipulates price up** (e.g., ETH price goes from $1,000 → $10,000).
    
2. Calls `mintDsc()` and mints the maximum allowed DSC based on the inflated price.
    
3. **Price reverts** back to normal shortly after (manipulation ends).
    
4. Attacker now has **excess DSC**, collateral no longer backs it → health factor < 1.


#### 🔸 Frontrunning Opportunity:

If another user submits a `mintDsc()` txn and attacker sees it in the mempool:

- Attacker manipulates price **before that txn gets mined**.
    
- Their txn gets priority (via bribe or higher gas).
    
- Victim mints at manipulated price and is **unknowingly undercollateralized**.




Written some findings at [[Findings]] 



I basically searched solodit for bugs on algo-stables and found some bugs that related to this protocol and I was able to find similar bugs in the protocol the deposit and minting flow had front running tendencies  

Also since oracle price cannot be trusted as the real-time price this can lead to price discrepancy  for example if the market price of the collateral is lower than the Oracle price it is possible to mint DSC by using the collateral and selling it to WETH basically eroding the stability of the DSC