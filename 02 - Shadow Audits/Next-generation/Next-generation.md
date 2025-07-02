#shadow-audit

# Thoughts

Next generation is a protocol that came up with an upgradable ERC-20 called EURF
the forwarder contract is designed to support gasless transaction that means the user signs the transaction off-chain and then a relayer, carries thi signed message and pays the gas and submits it to a smart contract that verifies the signature in turn this relayer  is paid in tokens.

## Contracts

### Token.sol

The **Token** contract is the main ERC20 implementation of EURF.

### Forwarder.sol

The **Forwarder** contract is designed to support gasless transactions. It forwards signed transactions that are executed on-chain, allowing users to interact with the EURF token without having to pay gas fees directly. This feature is especially useful for improving user experience and onboarding new users by abstracting the complexities of paying for gas.

  

### FeesHandlerUpgradeable.sol

The **FeesHandlerUpgradeable** contract manages the fee mechanisms for EURF. There are two types of fees: the **transaction fee (txfee_rate)** and the **gasless base fee**. The transaction fee is applied to every transfer, while the gasless base fee is used when transactions are forwarded by trusted forwarders. This contract ensures that fees are collected properly and the EURF ecosystem remains compliant.

  

### ERC20MetaTxUpgradeable.sol

The **ERC20MetaTxUpgradeable** contract extends the basic ERC20 functionality by adding support for meta-transactions. It allows users to sign approvals and other transactions off-chain, which can be forwarded by another party on-chain. This approach enables gasless transactions, where the transaction costs are borne by a different party (e.g., the dApp or the protocol).

  

### ERC20ControlerMinterUpgradeable.sol

The **ERC20ControlerMinterUpgradeable** contract manages minting permissions in a controlled manner. It allows the administrator to add or remove minters and to set minting limits, ensuring that the supply of EURF remains within predefined bounds. This contract helps enforce governance policies over token minting.

  
### ERC20AdminUpgradeable.sol

The **ERC20AdminUpgradeable** contract handles administrative functions for the EURF token. It allows for assigning administrative roles, controlling fees, managing blacklists, and overseeing upgrades of other contracts in the system. The use of upgradeable patterns ensures that the contract can be securely updated as needed.

The contracts use OpenZeppelin's **UUPSUpgradeable** pattern, which provides upgradability through proxies, allowing for future modifications without redeploying the entire contract.


## Findings
## [M-01] ERC-20 allowance bypass: spender can force sender to pay extra fees beyond approved amount]

###  description and impact

The `EURFToken` contract includes a transaction fee mechanism where a percentage of the transferred amount is deducted as a fee. This fee is deducted from the sender’s balance additionally. However, in the `transferFrom` function, which allows **spender addresses** to transfer tokens on behalf of the actual owner, the contract does not account for the fact that the total deducted amount (including fees) may exceed the approved allowance.

Specifically, if a user **approves a spender to transfer 100 tokens**, and the transaction fee is 10%, the spender will effectively cause the owner to **lose 110 tokens** (100 for the transfer and 10 for the fee); even though the allowance was only 100. This violates the ERC-20 standard, where the spender should only be able to transfer up to the approved amount.

```js
function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
    transferSanity(sender, recipient, amount);
    return super.transferFrom(sender, recipient, amount);
}
```

- The function calls `transferSanity(sender, recipient, amount)`, which **deducts additional fees** without checking if `amount + fee` exceeds the allowance.
- The actual deduction happens in `_payTxFee(sender, amount)`, which is triggered before executing the transfer:

```js
function transferSanity(address sender, address recipient, uint256 amount) internal {
    adminSanity(sender, recipient);
    if (_txfeeRate > 0) _payTxFee(sender, amount);
}
```

- `_payTxFee(sender, amount)` calculates the transaction fee **and deducts it directly from the sender’s balance**, without ensuring that the allowance covers it:

```js
function _payTxFee(address from, uint256 txAmount) internal override {
    uint256 txFees = calculateTxFee(txAmount);
    if (balanceOf(from) < txFees + txAmount) revert BalanceTooLow(txFees + txAmount, balanceOf(from));
    if (_feesFaucet != address(0)) _update(from, _feesFaucet, txFees);
    emit FeesPaid(from, txFees);
}
```

### [](https://code4rena.com/reports/2025-01-next-generation#proof-of-concept)Proof of Concept

The PoC built on Foundry framework to show that the decreased balance exceeds the approved allowance in `transferFrom` function.

```js
    function test_transferFrom_exceedAllowance() public {
        vm.startPrank(alice);
        token.approve(address(this), 100e6);
        vm.stopPrank();
        uint256 balBefore = token.balanceOf(alice);
        console.log("calling transferFrom to transfer 100 EURF");
        token.transferFrom(alice, bob, 100e6);
        assertGe(balBefore - token.balanceOf(alice), 100e6);
    }
```

Test result:

```
[PASS] test_transferFrom_exceedAllowance() (gas: 87670)
Logs:
  calling transferFrom to transfer 100 EURF

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.62ms (867.42µs CPU time)

Ran 1 test suite in 138.69ms (6.62ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended mitigation steps

Modify `transferFrom` to ensure that the spender’s allowance covers both the **transfer amount and fees**:

```js
function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
    uint256 txFees = calculateTxFee(amount);
    uint256 totalCost = amount + txFees;

    require(allowance(sender, msg.sender) >= totalCost, "ERC20: transfer amount exceeds allowance");

    transferSanity(sender, recipient, amount);
    return super.transferFrom(sender, recipient, amount);
}
```

With this update, **Bob can only transfer 100 EURF if Alice explicitly approves at least 110 EURF**, preventing unintended token loss.


## [M-02] Approve operation is not overridden to call `transferSanity`, thus its allowed to approve blacklisted accounts, which breaks protocol invariant 

In the readme invariant docs it is specified, as requirement for approval operations, the following conditions:

- Owner not being blacklisted
- Spender not being blacklisted


However, this invariant is broken by the protocol’s implementation, as there is no logic to enforce that. The `approve()` function is not overridden and `transferSanity()` is not called for the operation.

### Proof of Concept

To confirm we can refer to `Token.sol`:

The `approve()` function is not overridden.

### Recommended mitigation steps

Override the `approve` function and check if the `_msgSender()` and `spender` are not blacklisted, before allowing the operation. This can be done with `transferSanity()` check, the same way it is already implemented for the other operations.

```diff
contract EURFToken is
    ERC20MetaTxUpgradeable,
    ERC20AdminUpgradeable,
    ERC20ControlerMinterUpgradeable,
    FeesHandlerUpgradeable,
    UUPSUpgradeable
{
+    function approve(address spender, uint256 value) public override returns (bool) { 
+        transferSanity(_msgSender(), spender, value);
+        return super.approve(spender, value);
+    }
```



## [H-01] Cross-chain signature replay attack due to user-supplied `domainSeparator` and missing deadline check

### Finding description

The `_verifySig` function in `Forwarder.sol` accepts the `domainSeparator` as a user-provided input instead of computing it internally. This introduces a vulnerability where signatures can be replayed across different chains if the user’s nonce matches on both chains.

Additionally, there is no deadline check, meaning signatures remain valid indefinitely. This lack of expiration increases the risk of signature replay attacks and unauthorized transaction execution.

### Impact

1. **Cross-Chain Signature Replay Attack** – An attacker can reuse a valid signature on a different chain where the user’s nonce is the same, potentially leading to unauthorized fund transfers.
2. **Indefinite Signature Validity** – Without a deadline check, an attacker could store valid signatures and execute them at any point in the future.

### Proof of Concept

**Affected Code**

1. `_verifySig` function (user-controlled `domainSeparator` and no deadline check):

```js
function _verifySig(
    ForwardRequest memory req,
    bytes32 domainSeparator,
    bytes32 requestTypeHash,
    bytes memory suffixData,
    bytes memory sig
) internal view {
    require(typeHashes[requestTypeHash], "NGEUR Forwarder: invalid request typehash");
    bytes32 digest = keccak256(
        abi.encodePacked("\x19\x01", domainSeparator, keccak256(_getEncoded(req, requestTypeHash, suffixData)))
    );
    require(digest.recover(sig) == req.from, "NGEUR Forwarder: signature mismatch");
}
```

2. `_getEncoded` function (included in the signature computation):

```js
function _getEncoded(
    ForwardRequest memory req,
    bytes32 requestTypeHash,
    bytes memory suffixData
) public pure returns (bytes memory) {
    return abi.encodePacked(
        requestTypeHash,
        abi.encode(req.from, req.to, req.value, req.gas, req.nonce, keccak256(req.data)),
        suffixData
    );
}
```

3. `execute` function (where `_verifySig` is used):

```js
function execute(
    ForwardRequest calldata req,
    bytes32 domainSeparator,
    bytes32 requestTypeHash,
    bytes calldata suffixData,
    bytes calldata sig
) external payable returns (bool success, bytes memory ret) { 
    _verifyNonce(req);
    _verifySig(req, domainSeparator, requestTypeHash, suffixData, sig);
    _updateNonce(req);

    require(req.to == _eurfAddress, "NGEUR Forwarder: can only forward NGEUR transactions");

    bytes4 transferSelector = bytes4(keccak256("transfer(address,uint256)"));
    bytes4 reqTransferSelector = bytes4(req.data[:4]);

    require(reqTransferSelector == transferSelector, "NGEUR Forwarder: can only forward transfer transactions");

    (success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
    require(success, "NGEUR Forwarder: failed tx execution");

    _eurf.payGaslessBasefee(req.from, _msgSender());

    return (success, ret);
}
```

### Exploitation scenario

1. A user signs a valid transaction on Chain A.
2. An attacker copies the signature and submits it on Chain B (where the user has the same nonce).
3. Since `domainSeparator` is not verified by the contract, the signature is accepted on both chains.
4. The attacker successfully executes a transaction on Chain B without the user’s consent.

**Additional Risk:** The absence of a deadline check means an attacker could store a signature indefinitely and execute it at any time in the future.

### Recommended mitigation steps

1. **Compute `domainSeparator` On-Chain:** Instead of accepting `domainSeparator` as a function argument, compute it within the contract using `block.chainid` to ensure domain uniqueness.
2. **Enforce a Deadline Check:** Introduce a deadline verification within `_verifySig` to ensure signatures expire after a reasonable time. Example:
    
    ```js
    if (block.timestamp > req.deadline) revert DeadlineExpired(req.deadline);
    ```
    
3. **Use Chain-Specific Identifiers:** Incorporate `block.chainid` into the domain separator computation to prevent cross-chain signature reuse.

By implementing these fixes, the contract can prevent signature replay attacks and unauthorized transactions across different chains.