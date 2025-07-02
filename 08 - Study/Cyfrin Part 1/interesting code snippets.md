
Using mappings to increment a value so if called by 2 people it returns a mismatch 

    => usually used to to protect against making the same request twice 
    
```js
require(nonces[req.from]++ == req.nonce, "Nonce mismatch");
```

- `nonces` is a `mapping(address => uint256)` — it tracks the nonce for each user (`req.from`).
    
- `nonces[req.from]++` returns the **current value** and then **increments** it (post-increment).
    
- `req.nonce` is the nonce **signed by the user** in their meta-transaction.
    

This ensures that:

- The transaction is only accepted **if the nonce is exactly what the user signed** (meaning they haven’t sent it before).
    
- After accepting, it increments their nonce so it can’t be reused.



## 📊 Example Walkthrough

Let’s say:

solidity

CopyEdit

`nonces[alice] = 5; req.nonce = 5;`

Then:

solidity

CopyEdit

`nonces[alice]++ == req.nonce  // => 5 == 5 ✅`

Now, `nonces[alice]` becomes `6`.

If someone tries to submit the same request again:

solidity

CopyEdit

`nonces[alice]++ == req.nonce  // => 6 == 5 ❌`

It reverts with `"Nonce mismatch"`.