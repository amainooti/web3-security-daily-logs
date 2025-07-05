
## Quick Mental Model

Foundry runs tests in a local VM by default (not connected to any chain).
If your tests use mainnet contracts (like real WETH, Uniswap V2/V3),
then you must run with `--fork-url` so the mainnet state is available.

 