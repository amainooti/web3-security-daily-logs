

## Flow



```
depositCollateralAndMintDsc() -> depositCollateral()
							  -> mintDSC()
	

```


## Functions

```js
// q? Does this actually check allowed tokens?
modifier isAllowedToken(address token) {

if (s_priceFeeds[token] == address(0)) { //q? <@ Isn't this a zero address check?

revert DSCEngine__NotAllowedToken();

}

_;

}
```

 TODO Revisit this modifier
 


If the price feed is valid it returns else it returns a zero address hence the check
TODO: validate this later



```js
function _revertIfHealthFactorIsBroken(address user) internal view {

uint256 userHealthFactor = _healthFactor(user);

if (userHealthFactor < MIN_HEALTH_FACTOR) {

revert DSCEngine__BreaksHealthFactor(userHealthFactor);

}

}
```


Observation

`MIN_HEALTH_FACTOR = 1e18`   this represents 1.0 (Fixed point arithmetic's)
