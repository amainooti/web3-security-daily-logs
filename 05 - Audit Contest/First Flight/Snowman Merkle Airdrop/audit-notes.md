# [H-01] Overpayment Bug Allows Double Charging of ETH and WETH

#### Impact: High
### Likelihood: Medium

TODO: Write description


POC:

User Pays excess eth normal human everyday error and then the `Snow::buySnow()` function processes this transaction checks the first conditional and  fails it quickly goes on to the else block if the user has approved; if the user has approved weth this single fact causes the user to lose both eth and weth tokens.

Copy this block into `Snow.t.sol`

```js
function testOverpayingEthAndApprovingWeth() public {

uint256 required = FEE * 1;

uint256 overpayment = required + 1 ether;

  

// Fund the user

deal(victory, overpayment);

deal(address(weth), victory, overpayment);

  

// Approve WETH and send extra ETH

vm.startPrank(victory);

weth.approve(address(snow), required);

snow.buySnow{value: overpayment}(1);

vm.stopPrank();

  


assertEq(weth.balanceOf(address(snow)), required, "WETH was pulled");

assertEq(address(snow).balance, overpayment, "ETH was accepted");

assertEq(snow.balanceOf(victory), 1, "User received correct Snow tokens");

assertEq(weth.balanceOf(victory), overpayment - required, "WETH reduced");

assertEq(victory.balance, 0, "ETH not refunded");

}
```


the output:
```
 [26927] MockWETH::approve(Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], 5000000000000000000 [5e18])
    │   ├─ emit Approval(owner: victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8], spender: Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], value: 5000000000000000000 [5e18])
    │   └─ ← [Return] true
    ├─ [113296] Snow::buySnow{value: 6000000000000000000}(1)
    │   ├─ [32255] MockWETH::transferFrom(victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8], Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], 5000000000000000000 [5e18])
    │   │   ├─ emit Transfer(from: victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8], to: Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1], value: 5000000000000000000 [5e18])
    │   │   └─ ← [Return] true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8], value: 1)
    │   ├─ emit SnowBought(buyer: victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8], amount: 1)
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [1240] MockWETH::balanceOf(Snow: [0x90193C961A926261B756D1E5bb255e67ff9498A1]) [staticcall]
    │   └─ ← [Return] 5000000000000000000 [5e18]
    ├─ [0] VM::assertEq(5000000000000000000 [5e18], 5000000000000000000 [5e18], "WETH was pulled") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(6000000000000000000 [6e18], 6000000000000000000 [6e18], "ETH was accepted") [staticcall]
    │   └─ ← [Return] 
    ├─ [1284] Snow::balanceOf(victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8]) [staticcall]
    │   └─ ← [Return] 1
    ├─ [0] VM::assertEq(1, 1, "User received correct Snow tokens") [staticcall]
    │   └─ ← [Return] 
    ├─ [1240] MockWETH::balanceOf(victory: [0x299Fce47cC67d70efa97dD1880fc8C7a6CbAD2a8]) [staticcall]
    │   └─ ← [Return] 1000000000000000000 [1e18]
    ├─ [0] VM::assertEq(1000000000000000000 [1e18], 1000000000000000000 [1e18], "WETH reduced") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(0, 0, "ETH not refunded") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.97ms (1.81ms CPU time)
```


## Recommendations

Include a refund mechanism for when the user sends more eth than expected the below code accounts for when user sends below expected amount and above expected amount gracefully handling both scenarios. 
```diff
function buySnow(uint256 amount) external payable canFarmSnow {
- if (msg.value == (s_buyFee * amount)) {

-_mint(msg.sender, amount);

-}

+	uint256 totalCost = s_buyFee * amount;

+    if (msg.value > 0) {
+        if (msg.value < totalCost) revert S__InsufficientETH();
+        if (msg.value > totalCost) {
+            // Refund excess
+            (bool success, ) = msg.sender.call{value: msg.value - totalCost}("");
+            require(success, "Refund failed");
        }
+    } else {
+        i_weth.safeTransferFrom(msg.sender, address(this), totalCost);
+    }

    _mint(msg.sender, amount);
+    s_lastEarned[msg.sender] = block.timestamp; // better per-user cooldown
   emit SnowBought(msg.sender, amount);
+}

```



# [H-02] Global timer prevents all users from earning independently
#### Impact: High

#### Likelihood: High

## Description

The `Snow.earnSnow()` function relies on a **global cooldown timer** (`s_earnTimer`) to restrict how often users can claim Snow tokens. However, this timer applies to **all users**, meaning:

- When **any** user claims their weekly `Snow` token,
    
- It **resets the global timer** `s_earnTimer = block.timestamp;`,
    
- Which causes **all other users** to be blocked from earning until **that user's cooldown** expires.
    

This violates the expectation that earning is **per-user**, not globally rate-limited. It creates a **Denial of Service vector** where a single actor can delay or grief others.

POC:
TODO: write procedure of this
```js
function testGlobalTimerBlocksFarmers() public {

vm.prank(ashley);

snow.earnSnow();

assertEq(1, snow.balanceOf(ashley));

  

vm.prank(jerry);

vm.expectRevert(); // Because s_earnTimer is global, not per-user

snow.earnSnow();


}
```


# [H-03] Missing access control allows arbitrary minting of Snowman NFTs  
  
**Impact:**    
The `mintSnowman()` function lacks access control, enabling **anyone** to mint unlimited NFTs without meeting the protocol’s requirements (holding $SNOW and passing the airdrop check). This entirely breaks the fairness and security model of the Snowman system.  
  
**PoC:**  
```js  
nft.mintSnowman(alice, 1000); // no signature, no proof — works  
```  
  
**Recommended Mitigation:**    
Restrict minting to only be callable by the `SnowmanAirdrop` contract:  
  
```js  
require(msg.sender == address(i_airdrop), "Unauthorized");`  
```  
**Severity:** High

## Recommendation

consider using`mapping(address => uint256) s_lastClaimTime`


# [M-02] Claims Fail if User Balance Changes After Merkle Tree Generation

## description
TODO

POC:
TODO


```js
// This inadvertedly failed due to the fact that the merkle leaf hashes amount by dynamic balanceOf() on claim and if the amount changes claim fails

function testClaimFailIfUserBalanceChanges() public {

uint256 amountRequired = 2; // <@ Here because alice balance was incremented by 1 she becomes inelligible for airdrop

  

deal(address(snow), alice, amountRequired);

// Setup

assert(nft.balanceOf(alice) == 0);

vm.prank(alice);

snow.approve(address(airdrop), 1);

  

// Sign the first message

bytes32 digest = airdrop.getMessageHash(alice);

(uint8 v, bytes32 r, bytes32 s) = vm.sign(alKey, digest);

  

// First claim by satoshi

  

vm.prank(satoshi);

vm.expectRevert(SnowmanAirdrop.SA__InvalidProof.selector);

airdrop.claimSnowman(alice, AL_PROOF, v, r, s);

}
```

## Recommendation

**Observation:**  
Hashing the `receiver` address and their `balanceOf()` when generating the Merkle leaf unintentionally acted as a mechanism to prevent multiple claims, since any balance change would cause the leaf hash to become invalid.

**Issue:**  
However, this also introduces a critical flaw: if a user’s `balanceOf()` changes between snapshot generation and claim (e.g., due to receiving tokens), their Merkle proof becomes unverifiable — effectively locking them out of the claim.

As a result, the line `s_hasClaimedSnowman[receiver] = true;` is redundant, since a second claim attempt with a mismatched balance would already fail due to the hash mismatch. Yet, the lack of a proper state update (like setting a `claimed` flag) also means users are not explicitly prevented from retrying — they’re just implicitly blocked by signature/balance mismatch.

**Recommended Fix:**

- Decouple Merkle proofs from live on-chain state by snapshotting off-chain and using static `snapshotAmount` in proof validation.
    
- Explicitly track claims using `s_hasClaimedSnowman[receiver] = true` and reject duplicates using a proper check:
-
```diff

function claimSnowman(
    address receiver,
    bytes32[] calldata proof,
+    uint256 snapshotAmount, // <@ inlcude ths snaphot amount
    uint8 v, bytes32 r, bytes32 s
) external nonReentrant {

   // Zero Address checks
	// @> include previouss checks

+	require(s_hasClaimedSnowman[receiver], "Already claimed"); //<@ include this line to prevent multiple claims

- uint256 amount = i_snow.balanceOf(receiver);

+    bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, snapshotAmount))));
    
 if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {

revert SA__InvalidProof();

}

s_hasClaimedSnowman[receiver] = true; 
    
    // and mint or transfer NFT here
    ...
}

```

Also;
```diff
+ function getMessageHash(address receiver, uint256 snapshotAmount) public view returns (bytes32) {

if (i_snow.balanceOf(receiver) == 0) {

revert SA__ZeroAmount();

}

-  uint256 amount = i_snow.balanceOf(receiver);


return _hashTypedDataV4(

+ keccak256(abi.encode(MESSAGE_TYPEHASH, SnowmanClaim({receiver: receiver, amount: snapshotAmount})))

);

}
```


`snapshotAmount` should be the user’s token balance at the time of snapshot, captured off-chain and embedded into the Merkle tree