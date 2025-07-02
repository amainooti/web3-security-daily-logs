
## Observations

- Its an upgradeable contract that makes use of UUP's
- Its a tiered staking contract with different APY (annual percentage yield) rates based on staking position
- It implements a first-come-first-serve tiered system with different rewards
- The tier system is not a plan that users subscribe to it's first come first serve as stated above
	- The first 20% of stakers get the tier 1 package
	- The first 30% of stakers get the tier 2 package
- Admin can change the APY rates for all tiers (tier1APY, tier2APY, tier3APY => state variables)
- It only accepts the staking token meaning native ETH is not allowed (blocked by the receive function making sure the `msg.sender == address(stakingToken`) )



## Current Understanding

1. User deposits an amount of token
2. The interest is updated first of all (why is this done tho)
3. Theres a check for isNative which was set by the protocol to false
   - if its not true they call `transferFrom()` function and collect erc20 token
   - if the issNative is `true` then there's a typecast to weth and eth is deposited into
   the protocol.
1. Your deposited balance is then checked if its lower than the min amount to deposit(3000e18) and if it is then your'e calculated as out of tree and that counter is incremented `stakerCountOutOfTree++` user is also set to active and then the `user.outOfTree = true` and then your history is updated
2. If your amount is greater or equals to the min deposit youre assigned a joinId `user.joinId = nextJoinId++;` then you're included in the tree and your rank is updated and  the counter for in tree is increased`stakerCountInTree++` 
3. 