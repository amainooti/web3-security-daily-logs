# 2025-07-02

## Understanding Function Signature and Execution

1. **Function signature**  
    A function selector is created when the function signature is hashed, i.e., `updateHorseNumber(uint256)` — this is a function signature. The hash of this gives a `bytes4` + input data, e.g., `0xcdfead2e...`, usually a long string. The first 4 bytes of that long string is the function selector.
    
2. **Function execution**  
    How Solidity knows to execute a function call is like this: it first examines the first 4 bytes (function selector). This allows it to determine the actual function we want to execute. This is known as _function dispatching_, and it's something that Solidity does natively for us. So, in essence, in the EVM, this is when a smart contract uses the first 4 bytes of `calldata` to determine which function (which is just a group of opcodes) to send the `calldata` to.
    


---

### So, the EVM mainly has 3 core components we can use to “put stuff”:

1. **Stack**
    
2. **Memory** (volatile)
    
3. **Storage** (persistent)
    

Other components include **available gas**, **program counter**, and **transient storage** (not a thing — yet).

- The **cheapest** place to do stuff is the stack. Doing things like `ADD`, etc.  
    The EVM is known as a **stack machine**, and it's the main data structure we work with in Ethereum.
    
- **Stack**: The stack is a **LIFO** data structure — Last In, First Out.
    
- **Memory**: Unlike the stack, where things get pushed and popped, in memory you can stick anything anywhere (as long as that position is free). At the end of execution, it vanishes — _poof_. That's why it's said to be volatile.
    
- **Storage**: Similar to memory in how data is placed, but differs in that it **retains** data. This data persists and is the most **costly** data structure in the EVM.


---

## Examining the Huff Language as an Entry Point to Understanding the EVM

**Task**

1. Create a function dispatcher in Huff.
    
    When we compile smart contract code, most smart contracts are compiled into 3 or 4 sections:
    
    1. The contract creation code
        
    2. Runtime code
        
    3. Metadata
        
    
    The Solidity compiler gives a delimiter or a separation to determine these sections — it adds an `INVALID` opcode for us to use to mark these sections.
    
    - **Contract creation bytecode**: Essentially just tells the compiler to take the binary after it and stick it on-chain.  
        Example: `60008060093d393df3` — when examining the contract creation opcode, you can identify it from the `39` (`CODECOPY`) opcode.
        
        > Note: We will return to these later.
        
    
    Now that we kind of have an idea about how a compiled contract looks and how things are saved in the EVM, let’s go right back to our goal.

### 🧠 Extracting Function Selector from Calldata (Using `calldataload` and `shr`)

### 🔹 Step 1: `calldataload` Only Loads 32 Bytes

The `calldataload` opcode **always loads 32 bytes** (i.e., 256 bits) from calldata, starting at the offset you provide.  
So, when you run:

`0x00 calldataload`

You're loading `calldata[0:32]` — that's 32 bytes total, even if all you care about is the first **4 bytes** (the function selector).

---

### 🔹 Step 2: Truncating the Remaining 28 Bytes

We only want the **first 4 bytes** of that 32-byte word — the **function selector**.  
The rest (28 bytes) is just arguments or padding, which we don’t care about **at this point**.

To get rid of those extra 28 bytes, we need to **truncate** them — and that’s where the `shr` (shift right) opcode comes in.

---

### 🔹 Step 3: Use `shr` (Logical Right Shift)

The `shr` opcode is like the `>>` operator in high-level languages.  
It shifts the value on the stack **to the right**, throwing away the least significant bits.

So to isolate just the first 4 bytes, we want to shift off the remaining 28 bytes.

---

### 🔹 Step 4: Calculate Shift Amount in Bits

We’re working in **bytes**, but the EVM operates in **bits** — so we need to convert:

- We have **32 bytes** total
    
- We want to keep **4 bytes**
    
- That means we need to shift off **28 bytes**
    

Now convert to bits:


`8 bits = 1 byte x bits = 28 bytes  → x = 8 × 28 = 224 bits`

So, we want to shift the word **right by 224 bits**.

---

### 🔹 Step 5: Use `shr` with 224

In Huff:

`0x00 calldataload 0xe0        // 224 in hex shr         // shift right by 224 bits`

Now the stack has only:

`[e026c017] ← your 4-byte function selector`

---

### 🧰 Bonus: Use `cast` to Convert Decimal to Hex

Want to convert the shift amount (224) to hex?t

`cast to-hex 224 # Output: 0xe0`

Want to check the hex back to decimal?

`cast to-dec 0xe0 # Output: 224`

---

### ✅ Summary

- `calldataload` loads 32 bytes (256 bits)
    
- We only want the first 4 bytes (selector)
    
- To truncate the remaining 28 bytes, we right shift by `8 * 28 = 224 bits`
    
- `shr 224` achieves this
    
- You can use `cast` to get hex for the shift value: `cast to-hex 224 → 0xe0`


# 2025-07-03

### Steps in creating a function dispatcher
```
#define macro MAIN() = takes(0) returns(0) {
    // Step 1: Load selector from calldata
    // -----------------------------------
    0x00                // PUSH1 0x00 → stack: [0x00]
    calldataload        // → stack: [0xe026c017000...0001] (32 bytes)
    0xe0                // PUSH1 0xe0 → stack: [0xe0, full_calldata]
    shr                 // shift right 224 bits → stack: [0xe026c017] ✅

    // Step 2: Compare to first selector (updateHorseNumber)
    // ------------------------------------------------------
    dup1                // DUP1 → copy selector → stack: [0xe026c017, 0xe026c017]
    0xcdfead2e          // PUSH4 0xcdfead2e → stack: [0xcdfead2e, 0xe026c017, 0xe026c017]
    eq                  // EQ pops top 2 → compares → false (0) → stack: [0, 0xe026c017]
    updateJump          // PUSH2 <addr> → stack: [updateJump, 0, 0xe026c017]
    jumpi               // JUMPI pops top 2 → condition = 0, doesn't jump → stack: [0xe026c017]

    // Step 3: Compare to second selector (readNumberOfHorses)
    // --------------------------------------------------------
    dup1                // DUP1 → stack: [0xe026c017, 0xe026c017]
    0xe026c017          // PUSH4 0xe026c017 → stack: [0xe026c017, 0xe026c017, 0xe026c017]
    eq                  // EQ pops top 2 → compares → true (1) → stack: [1, 0xe026c017]
    readJump            // PUSH2 <addr> → stack: [readJump, 1, 0xe026c017]
    jumpi               // JUMPI pops top 2 → condition = 1, jumps to readJump → stack: [0xe026c017]

    // Step 4: At jump destination (readJump:)
    // ----------------------------------------
    readJump:
        pop             // Remove selector from stack → stack: []
        READ_NUMBER_OF_HORSES()
}

```

## Takeaways

1. Operations are done in the stack and then sent to either memory or storage 
2. The EVM is made of this opcodes which help manipulate data
3. Usually Order of operation matters when something is pushed to the stack the opcode
   that intends to manipulate it will likely make use of the data at the top of it i.e 
   when we wanted to load call data the `CALLDATALOAD` opcode made use of the data on 
   on-top to determine the amount of offset it should load from `calldataload[0:32]` 