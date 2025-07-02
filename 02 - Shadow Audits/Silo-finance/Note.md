# Silo Finance Deep Dive: Withdraw Flow & Notes

## Protocol Overview

Silo Finance is a **non-custodial, DAO-governed lending protocol** built around the principle of **isolated markets** (silos). Each asset has its own silo to minimize systemic risk. Each silo uses **ERC4626 vaults** for managing deposits and withdrawals, with additional custom logic layered on top.

There are two relevant enums for accounting:

```
enum AssetType {
    Protected,   // non-borrowable deposits
    Collateral,  // borrowable deposits
    Debt         // borrowed tokens
}

enum CollateralType {
    Protected,   // non-borrowable, locked deposits
    Collateral   // borrowable deposits
}
```

## Withdraw Flow Summary

### User-Level Call

```
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares)
```

This calls the internal `_withdraw` with `shares = 0`, meaning the user is specifying the amount of `assets` they want to withdraw.

### `_withdraw` -> ERC4626 Logic

- Converts between `assets` and `shares`
    
- **If collateral type is Collateral**, checks solvency first via `_checkSolvencyWithoutAccruingInterest`
    
- Checks available **liquidity**
    
- Updates internal storage: `totalAssets[type] -= assets`
    
- Calls hooks
    
- Burns share tokens
    
- Transfers underlying assets
    

### `SiloERC4626.withdraw()`

Full implementation (see below) checks:

1. That `totalSupply != 0`
    
2. Converts between `assets <-> shares`
    
3. Checks available **liquidity**
    
4. Updates internal storage: `totalAssets[type] -= assets`
    
5. Burns `shares`
    
6. Transfers tokens (if `_asset != address(0)`) after validating that protected balance is not violated
    

## Notes on Specific Functions

### `_checkSolvencyWithoutAccruingInterest`

This checks whether a user is solvent using:

```
SiloSolvencyLib.isSolvent(
    _collateralConfig,
    _debtConfig,
    _user,
    ISilo.AccrueInterestInMemory.No
);
```

If `_debtConfig.silo == address(0)`, the user has no debt and is solvent by default.

Otherwise, `isSolvent` calls `getLtv` → which eventually calls `calculateLtv` → which uses `getPositionValues` to compute collateral and debt values → computes LTV.

Then LTV is compared to `collateralConfig.lt` (liquidation threshold). If `ltv <= lt`, the user is solvent.

### `convertToAssetsOrToShares`

Handles flexible input — users may specify either shares or assets:

```
if (_assets == 0) {
    require(_shares != 0)
    shares = _shares;
    assets = convertToAssets(...);
} else if (_shares == 0) {
    shares = convertToShares(...);
    assets = _assets;
} else {
    revert();
}
```

This allows users to either withdraw a specific number of `assets` or `shares`, but not both.

### `liquidity()`

Returns how much is borrowable (from the DAO's perspective):

```
return _collateralAssets - _debtAssets;
```

The protocol considers DAO and deployer fee cuts as non-borrowable, so this removes `debtAssets` from `collateralAssets`.

### Hooks

Hooks are configured using bitmaps and the `|` bitwise OR operator. Example:

```
uint256 action = WITHDRAW | (collateralType == Collateral ? COLLATERAL_TOKEN : PROTECTED_TOKEN);
```

This lets the hook receiver selectively respond to only relevant actions.

### `SiloStorageLib.getSiloStorage()`

```
assembly {
    $.slot := _STORAGE_LOCATION
}
```

Returns storage struct pointer based on fixed storage slot (`bytes32`). This is used to fetch and update `totalAssets[AssetType]`.

This pattern allows libraries to work on storage without explicitly declaring variables in every contract. It is common in proxy-safe libraries.

### Burn and Transfer

After all logic checks:

```
IShareToken(_shareToken).burn(owner, spender, shares);
IERC20(_asset).safeTransfer(receiver, assets);
```

Note: `burn` checks allowances and may trigger a reentrant call, but reentrancy protection is assumed.

If `_asset == address(0)` → just burns the shares (used during collateral transitions).

### Protected Balance Check

```
require(
  protectedBalance == 0 ||
  balanceOf(address(this)) - assets >= protectedBalance,
  ISilo.ProtectedProtection()
);
```

Ensures you’re not accidentally draining assets below what's marked as protected.

---

## Final Understanding

- **Shares** represent claim on vault assets. They’re minted on deposit and burned on withdrawal.
    
- **Assets** are the actual ERC20 tokens.
    
- Vault uses **two types of accounting** (`Protected`, `Collateral`) which are distinguished using enums and bitmasking.
    
- Liquidity is calculated as: `collateralAssets - debtAssets`
    
- Storage access happens through a fixed slot using inline assembly.
    
- Hooks allow custom logic execution before and after actions.
    
- The `Action::withdraw` path specifically uses hooks and solvency checks for **Collateral** withdrawals only.
    
- `isSolvent()` ultimately compares user’s LTV to liquidation threshold `lt` in `collateralConfig`.
    

## Remaining Exploration

- DAO hooks and how they interact with share token logic.
    
- How interest and rewards are distributed.
    
- Governance and upgradability.
    
- Risk mitigation via isolation of silos.
    

Let me know when you’re ready to continue — next steps could be analyzing `deposit()` and DAO-level interest logic or trying to attack accounting invariants (e.g. improper liquidity checks, hook misuses, solvency edge cases).




Question


Ok final understanding from a high level of withdraw flow  is this
the main external withdraw calls _withdraw which passes the parameters as auguments for the action::withdraw, this function has a hook that first checks collateral type (wuthdrawAction) using bitmap (bitwise OR operator)  then calls a hook receiver with this action
It gets the withdrawal config which involves debtConfig, collateralConfig and depositConfig all the things used to check solvency and then if the deposit is collateral it then checks the solvency

So the action::withdraw's primary duty is to ensure that the caller is OK to withdraw funds, then the main withdrawal which is the erc4626 checks the liquidity, updates the total asset(hey someone just removed assets heres the update) it gets the assets and shares and then performs the burning and transfer of assets to the designated spender this also makes me wonder is the protected balance  a shared state? its like a global state like totalAsset? cuz in that function they really made known that the protected asset should not go below a certain level

we should talk comprehensively about this

```js
    
	/// @notice Withdraw assets from the silo
    /// @dev Asset type is not verified here, make sure you revert before when type == Debt
    /// @param _asset The ERC20 token address to withdraw; 0 means tokens will not be transferred. Useful for
    /// transition of collateral.
    /// @param _shareToken Address of the share token being burned for withdrawal
    /// @param _args ISilo.WithdrawArgs
    /// @return assets The exact amount of assets withdrawn
    /// @return shares The exact number of shares burned in exchange for the withdrawn assets
    function withdraw(
        address _asset,
        address _shareToken,
        ISilo.WithdrawArgs memory _args
    ) internal returns (uint256 assets, uint256 shares) {
        uint256 shareTotalSupply = IShareToken(_shareToken).totalSupply();
        require(shareTotalSupply != 0, ISilo.NothingToWithdraw());

        ISilo.SiloStorage storage $ = SiloStorageLib.getSiloStorage();

        ISilo.AssetType collateralType = ISilo.AssetType(uint256(_args.collateralType));

        {
            // Stack too deep
            uint256 totalAssets = $.totalAssets[collateralType];

            (assets, shares) = SiloMathLib.convertToAssetsOrToShares(
                _args.assets,
                _args.shares,
                totalAssets,
                shareTotalSupply,
                Rounding.WITHDRAW_TO_ASSETS,
                Rounding.WITHDRAW_TO_SHARES,
                collateralType
            );

            uint256 liquidity = _args.collateralType == ISilo.CollateralType.Collateral
                ? SiloMathLib.liquidity($.totalAssets[ISilo.AssetType.Collateral], $.totalAssets[ISilo.AssetType.Debt])
                : $.totalAssets[ISilo.AssetType.Protected];

            // check liquidity
            require(assets <= liquidity, ISilo.NotEnoughLiquidity());

            $.totalAssets[collateralType] = totalAssets - assets;
        }

        // `burn` checks if `_spender` is allowed to withdraw `_owner` assets. `burn` calls hook receiver
        // after tokens transfer and can potentially reenter, but state changes are already completed,
        // and reentrancy protection is still enabled.
        IShareToken(_shareToken).burn(_args.owner, _args.spender, shares);

        if (_asset != address(0)) {
            // does not matter what is the type of transfer, we can not go below protected balance
            uint256 protectedBalance = $.totalAssets[ISilo.AssetType.Protected];

            require(
                protectedBalance == 0 || IERC20(_asset).balanceOf(address(this)) - assets >= protectedBalance,
                ISilo.ProtectedProtection()
            );

            // fee-on-transfer is ignored
            IERC20(_asset).safeTransfer(_args.receiver, assets);
        }
    }
    ```


## Observation

Each user has an asset type depending on their position 

`mapping(AssetType assetType => uint256 assets) totalAssets;`

1. If a user Deposits funds his assetType is updated to collateral 
2. If a user Withdraws funds He is still at a collateralType but the total asset is updated and that asset is removed internally.
