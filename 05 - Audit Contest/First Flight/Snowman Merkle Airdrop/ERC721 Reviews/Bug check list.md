1. Reentrancy
2. Missing Access control
3. Replay attack (check things like requestId, are there things that can be reused to mint more nfts than you should)
4. Unbounded Mint cap (Causing the price of mint to increase per mint)
5. Mint logic flaws(`mint(msg.sender, tokenId)` where the original owner was not remembered)
6. Unchecked state's  (Check if states are updated)
7. Math logic and protocol logic