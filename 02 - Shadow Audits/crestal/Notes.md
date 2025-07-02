## `Payment.sol` 
This contact possesses the `Payment::checkNFTOwnership()` which accepts `nftAddress` ,  `nftId` and the `userAddress` this checks for zero address and then type converts the `nftAddress` to an ERC-721 and uses the `ownerOf` function to verify if the `nftId` matches the `userAddress`.


```js
function checkNFTOwnership(address nftTokenAddress, uint256 nftId, address userAddress)

public

view

returns (bool)

{

// e! check for zero address

require(nftTokenAddress != address(0), "Invalid NFT token address");

require(userAddress != address(0), "Invalid user address");

  

IERC721 nftToken = IERC721(nftTokenAddress); // Type casting the nftTokenAddress

  

// Checks if the user owns the NFT

return nftToken.ownerOf(nftId) == userAddress;

}
```


It also consists of `Payment::payWithERC20` this transfers an ERC-20 token using the `safeTransferFrom` function it accepts the address of the token, from and to (typical `safeTransferfrom`).



## `EIP712.sol`
This contract is an upgradable contract inheriting the EIP712Upgradeable contract from openzepplin.

It has 2 constants 
```js
bytes32 public constant PROPOSAL_REQUEST_TYPEHASH =

keccak256("ProposalRequest(bytes32 projectId,string base64RecParam,string serverURL)");

bytes32 public constant DEPLOYMENT_REQUEST_TYPEHASH =

keccak256("DeploymentRequest(bytes32 projectId,string base64RecParam,string serverURL)");
```

Proposal request and a deployment request which posses the same parameters

TODO: find out what the parameters are what that base64RecParam, serverURL and projectId means 

**Hypothesiss**
This looks like a governance protocol? 

---

the next function is a getter that fetches the contract address from EIP712 Domain seperator

```js
function getAddress() public view returns (address) {

(,,,, address verifyingContract,,) = eip712Domain(); //<@ provides protection against replay attacks

return verifyingContract;

}
```

The rest gets the digest of the deployment and the proposal and then finally a signer address is gotten using the message hash and the packed encoding of the signature component