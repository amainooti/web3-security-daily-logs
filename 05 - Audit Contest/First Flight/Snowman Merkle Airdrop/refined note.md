# [H-01] Overpayment Bug Allows Double Charging of ETH and WETH

### Summary

The `Snow::buySnow()` function in the `Snow` contract accepts both ETH and WETH as payment but lacks proper refund handling for ETH overpayments. If a user mistakenly sends more ETH than required and has approved WETH, the contract deducts **both ETH and WETH**, resulting in double payment for a single purchase.

### Vulnerability Details

When a user overpays using ETH **and** has already approved WETH, the `buySnow()` logic enters the `else` branch and **pulls WETH**, while still accepting the full ETH payment. The function lacks a refund mechanism for overpayment, leading to unnecessary token loss:

```js
function buySnow(uint256 amount) external payable canFarmSnow {

if (msg.value == (s_buyFee * amount)) { // <@ this condition only checks if the user amount is equal to the total amount but does not account for over payments

_mint(msg.sender, amount);

} else {

i_weth.safeTransferFrom(msg.sender, address(this), (s_buyFee * amount));

_mint(msg.sender, amount);

}
```

This is exacerbated by the user’s natural tendency to overpay slightly (e.g., `+1 ether`)

### Impact

- Users can be **double-charged**: once in ETH, once in WETH.
    
- Lost ETH due to missing refund logic.
    
- Poor UX and exploitability in automated frontends or integrations.
    

### Tools Used

- Foundry tests (`forge test`)
    
- Manual code review
    

### PoC

Paste this into `Snow.t.sol`:
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

Output confirms both ETH and WETH were accepted.

### Recommended Mitigation

Refactor `buySnow()` to refund any overpaid ETH:

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
---

# [H-02] Global Timer Prevents Independent Earnings

### Summary

The Snow protocol incorrectly uses a **global cooldown timer** (`s_earnTimer`) for earnings. This design flaw makes **all users share the same timer**, enabling griefing or denial-of-service (DoS) by a single user.

### Vulnerability Details

When any user calls `earnSnow()`, the contract resets `s_earnTimer = block.timestamp`, affecting everyone. The expectation is that earning should be per-user (independent claim periods), but currently it blocks **everyone else** for the cooldown duration.

### Impact

- One user can block earnings for all others.
    
- Users are incentivized to grief or race to block others.
    
- Breaks UX and fairness.
    

### Tools Used

- Foundry
    
- Manual code inspection
    

### PoC

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
### Recommended Mitigation

Replace the global timer with a per-user mapping:


```diff
+ import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

+ contract Snow is ERC20, Ownable, ReentrancyGuard {
...
+ mapping(address => uint256) public s_lastClaimTime;
...

function earnSnow() external canFarmSnow noRentrant{
- if (s_earnTimer != 0 && block.timestamp < (s_earnTimer + 1 weeks)) {

- revert S__Timer();

+     if (s_lastEarned[msg.sender] != 0 && block.timestamp < (s_lastEarned[msg.sender] + 1 weeks)) {
+         revert S__Timer();

_mint(msg.sender, 1);

+ SnowEarned(msg.sender, 1)
+  s_lastEarned[msg.sender] = block.timestamp;

}

```

Update `earnSnow()` to check against `s_lastClaimTime[msg.sender]`.

---

# [H-03] Missing Access Control Allows Arbitrary Snowman NFT Minting

### Summary

The `Snowman::mintSnowman()` function is **externally callable with no access control**, allowing anyone to mint **unlimited NFTs**. This bypasses the protocol’s intended eligibility requirements and breaks the airdrop mechanism.

### Vulnerability Details

`mintSnowman()` lacks a `require` or `onlyRole` modifier. It should only be callable by the `SnowmanAirdrop` contract after validating Merkle proofs and signatures. The absence of this check makes the entire Snowman NFT reward system meaningless.

```js
function mintSnowman(address receiver, uint256 amount) external { // <@ No access control can be called by anyone and abused

for (uint256 i = 0; i < amount; i++) {

_safeMint(receiver, s_TokenCounter);

  

emit SnowmanMinted(receiver, s_TokenCounter);

  

s_TokenCounter++;

}

}
```
### Impact

- Any user can mint unlimited NFTs.
    
- Undermines the reward mechanism tied to Snow token holding.
    
- Ruins scarcity and value.
    

### Tools Used

- Manual review
    
- Foundry test
    

### PoC

```js
nft.mintSnowman(alice, 1000); // no signature, no proof — works  
```
### Recommended Mitigation


```diff
// >>> ERROR
...
+  error S__MaxSupplyReached();
+ uint256 public constant MAX_SUPPLY = 10_000;

...

function mintSnowman(address receiver, uint256 amount) external {
    if (msg.sender != address(i_airdrop)) revert S__Unauthorized(); // Access control
+
+     if (s_TokenCounter + amount > MAX_SUPPLY) revert S__MaxSupplyReached(); // Cap check
+
+     for (uint256 i = 0; i < amount; i++) {
+         _safeMint(receiver, s_TokenCounter);
+         emit SnowmanMinted(receiver, s_TokenCounter);
+         s_TokenCounter++;
+     }
```

---

# [M-01] Claims Fail if User Balance Changes After Merkle Tree Generation

### Summary

The Merkle leaf generation logic uses `balanceOf(receiver)` dynamically at claim time, which causes the leaf hash to become invalid if the user’s token balance has changed since the off-chain snapshot. This causes valid users to become ineligible.

### Vulnerability Details

The airdrop’s Merkle leaf is computed as:

```js
keccak256(abi.encode(receiver, i_snow.balanceOf(receiver)))
```

If a user sends/receives tokens after the Merkle tree was created, their balance will differ from the snapshot, and `MerkleProof.verify()` will fail.

This design indirectly tries to bind proofs to current holdings but instead creates a usability and UX failure mode.

### Impact

- Eligible users are unable to claim.
    
- No clear feedback about why claim failed.
    
- `s_hasClaimedSnowman[receiver] = true;` is redundant and ineffective.
    

### Tools Used

- Foundry
    
- Manual code trace
    

### PoC

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
### Recommended Mitigation

Snapshot user balances **off-chain** and use static `snapshotAmount` in both:

- The Merkle leaf
    
- The signature
    

Update `getMessageHash()`:
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

Update `claimSnowman()`:
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


# [L-01] Lack of `FeeCollected` Event Limits Transparency of Protocol Activity

### Summary

The `collectFee()` function transfers both WETH and native ETH to the designated collector address, but **fails to emit a `FeeCollected` event**. This omission reduces **on-chain transparency**, hampers **off-chain analytics**, and can make it harder for community members, indexers, or auditors to track fee activity.

---
### Vulnerability Details

Inside the `Snow` contract, the `collectFee()` function is designed to sweep accumulated protocol fees (WETH + ETH) to a collector wallet. However, there is no `FeeCollected` event emitted after the transfer is completed.

```js
function collectFee() external onlyCollector {
    uint256 collection = i_weth.balanceOf(address(this));
    i_weth.transfer(s_collector, collection);

    (bool collected,) = payable(s_collector).call{value: address(this).balance}("");
    
    require(collected, "Fee collection failed!!!");
}

```

### Impact

- Indexers (The Graph, Dune, etc.)
-  Security teams tracking fund flows

### Tools Used

- Manual review

### Recommended Mitigation

Emit a `FeeCollected` event at the end of the function, ideally including both the WETH and ETH amounts swep

```diff

...
+ event FeeCollected(address indexed caller, address indexed collector, uint256 wethAmount, uint256 ethAmount);

...
function collectFee() external onlyCollector {
    uint256 wethCollected = i_weth.balanceOf(address(this));
    i_weth.transfer(s_collector, wethCollected);

    uint256 ethCollected = address(this).balance;
    (bool collected,) = payable(s_collector).call{value: ethCollected}("");
    require(collected, "Fee collection failed!!!");

+   emit FeeCollected(msg.sender, s_collector, wethCollected, ethCollected);
}

```


