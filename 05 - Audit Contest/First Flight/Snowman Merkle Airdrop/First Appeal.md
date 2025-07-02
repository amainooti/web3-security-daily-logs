# Appeal For Double Payment Vulnerability

##  Summary

The dismissal states:

> _“If they don't send the exact amount of ETH, their WETH (exact amount) is used instead, and their ETH is untouched.”_

This is **factually incorrect**. My proof shows that:

>  **When a user overpays with ETH and has WETH approved, the contract:  
> (1) consumes WETH,  
> (2) retains the overpaid ETH,  
> (3) and never refunds the ETH.**

This results in a  double payment, not just a fallback to WETH.
## root cause

```js
function buySnow(uint256 amount) external payable canFarmSnow {
  

if (msg.value == (s_buyFee * amount)) { // <@ this does not account for overpayment 

_mint(msg.sender, amount);

} else {

i_weth.safeTransferFrom(msg.sender, address(this), (s_buyFee * amount));

_mint(msg.sender, amount);

}

s_earnTimer = block.timestamp;

  

emit SnowBought(msg.sender, amount);

}
```


## POC

**Analysis Of the Evidence**

1. ETH was sent `{value: 10}` shows 10 wei ETH was included in the transaction
2. WETH branch executed: The `transferFrom` call proves the function went to the else branch
3. Double payment occurs: If weth approval existed, both ETH(10 wei) and Weth 5e18 would be consumed

```js
function testDoublePaymentProof() public {

address victim = makeAddr("victim");

deal(victim, 1 ether); // Give victim ETH

deal(address(weth), victim, 10 ether); // Give victim WETH

  

vm.startPrank(victim);

weth.approve(address(snow), FEE); // Approve WETH

  

uint256 ethBefore = victim.balance;

uint256 wethBefore = weth.balanceOf(victim);

uint256 contractEthBefore = address(snow).balance;

  

// Send wrong amount of ETH - this should go to WETH branch

snow.buySnow{value: 10}(1); // 10 wei != FEE (5e18)

  

vm.stopPrank();

  

// Victim loses BOTH ETH and WETH

assertEq(victim.balance, ethBefore - 10, "Lost 10 wei ETH");

assertEq(weth.balanceOf(victim), wethBefore - FEE, "Lost FEE amount of WETH");

assertEq(address(snow).balance, contractEthBefore + 10, "Contract kept ETH");

assertEq(snow.balanceOf(victim), 1, "Only got 1 token");

  

// Total paid: 10 wei + 5e18 WETH, but should only pay 5e18

}
```

```
Snow::buySnow{value: 10}(1)
    │   ├─ [32255] MockWETH::transferFrom(victim: [0x131f15F1fD1024551542390614B6c7e210A911AF], Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], 5000000000000000000 [5e18])
    │   │   ├─ emit Transfer(from: victim: [0x131f15F1fD1024551542390614B6c7e210A911AF], to: Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], value: 5000000000000000000 [5e18])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: victim: [0x131f15F1fD1024551542390614B6c7e210A911AF], value: 1)
    │   ├─ emit SnowBought(buyer: victim: [0x131f15F1fD1024551542390614B6c7e210A911AF], amount: 1)
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(999999999999999990 [9.999e17], 999999999999999990 [9.999e17], "Lost 10 wei ETH") [staticcall]
    │   └─ ← [Return] 
    ├─ [1240] MockWETH::balanceOf(victim: [0x131f15F1fD1024551542390614B6c7e210A911AF]) [staticcall]
    │   └─ ← [Return] 5000000000000000000 [5e18]
    ├─ [0] VM::assertEq(5000000000000000000 [5e18], 5000000000000000000 [5e18], "Lost FEE amount of WETH") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(10, 10, "Contract kept ETH") [staticcall]
    │   └─ ← [Return] 
    ├─ [1284] Snow::balanceOf(victim: [0x131f15F1fD1024551542390614B6c7e210A911AF]) [staticcall]
    │   └─ ← [Return] 1
    ├─ [0] VM::assertEq(1, 1, "Only got 1 token") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.05ms (4.32ms CPU time)

Ran 1 test suite in 1.18s (8.05ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

4. Eth is trapped: There is no refund mechanism for the sent ETH