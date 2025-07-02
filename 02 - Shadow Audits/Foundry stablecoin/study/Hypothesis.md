
Key things we have to attempt to do

1. Find accounting edge cases to break

2. See how we can exploit this with flashloan 

3. front running, back running or sandwitch attacks


Tried to see if liquidators can grief users 

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


it failed `DSCEngine__HealthFactorNotImproved` 