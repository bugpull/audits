## [M-01] Incorrect Asset Valuation Leading to lower than expected Profits in Alchemix Strategy
### Summary

The `_harvestAndReport()` function incorrectly accounts for underlying WETH tokens at a 1:1 ratio with alETH, leading to inflated total asset calculations and incorrect profit reporting.

### Vulnerability Details

In `_harvestAndReport()`:

```
_totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
```

The function adds WETH (underlying) balance directly to alETH (asset) balance, but WETH:alETH trades at a premium (>1:1 ratio). This creates two issues:

1. Incorrect accounting by treating WETH 1:1 with alETH when it should be valued higher
    
2. The strategy should not hold WETH as it should be immediately swapped via `claimAndSwap()`
    

The correct calculation should be:

```solidity
_totalAssets = unexchanged + asset.balanceOf(address(this));
```

And WETH should be swapped to alETH immediately when claimed. (assuming that the commented Logic has been uncommented)

### Impact

Medium - This vulnerability leads to:

- deflated total assets reporting
    
- Incorrect profit calculations
    
- Wrong performance fee charges
    
- Inaccurate share price calculations
    
- Potential economic loss for users through incorrect share pricing
    

The impact is amplified because `report()` in TokenizedStrategy.sol uses this value for critical accounting including:

- Profit/loss calculations
    
- Fee distributions
    
- Share price updates
    
- Profit unlocking mechanics
    

### Tools Used

- Manual code review
    
- Code cross-referencing with TokenizedStrategy.sol
    
- Understanding of Alchemix protocol mechanics
    

### Recommendations

1. Remove underlying token from total assets calculation:
    

```solidity
function _harvestAndReport() internal override returns (uint256 _totalAssets) {
    uint256 claimable = transmuter.getClaimableBalance(address(this));
    
    if (claimable > 0) {
        transmuter.claim(claimable, address(this));
        // Immediately swap claimed WETH to alETH
        _swapUnderlyingToAsset(underlying.balanceOf(address(this)));
    }

    _totalAssets = transmuter.getUnexchangedBalance(address(this)) + 
                   asset.balanceOf(address(this));
}
```

2. Enforce immediate WETH to alETH swaps after claims
    
3. Add validation to ensure no WETH balance remains after operations
    
4. Consider adding a minimum premium check for WETH:alETH swaps
   
## [M-02] Incorrect Balance Deployed Calculation Leading to Profit Manipulation
### Summary

The `balanceDeployed()` function incorrectly includes underlying WETH tokens at face value and fails to account for exchanged balances, causing issues for off-chain calculations

### Vulnerability Details

In `balanceDeployed()`:

```solidity
function balanceDeployed() public view returns (uint256) {
    return transmuter.getUnexchangedBalance(address(this)) + underlying.balanceOf(address(this)) + asset.balanceOf(address(this));
}
```

two issues:

1. WETH is counted at face value when it trades at a premium to alETH
    
2. Exchanged balance (claimable WETH) is not included in calculation
    

The function should include:

- Unexchanged alETH balance
    
- Current alETH balance
    
- Exchanged balance (claimable)
    

### Impact

Low - This vulnerability affects core accounting:

Since this function is not called any where and used in off-chain logic only

### Tools Used

- Manual code review
    
- Cross-reference with Alchemix protocol documentation
    

### Recommendations

1. Include exchanged balance in calculations
    
```solidity
function balanceDeployed() public view returns (uint256) {
    return transmuter.getUnexchangedBalance(address(this)) + 
           asset.balanceOf(address(this)) +
           transmuter.getClaimableBalance(address(this));
}
```

2. Remove direct WETH accounting
   
## [M-03] Incorrect Asset Accounting in Harvest Report Leading to Share Price Manipulation
### Summary

The `_harvestAndReport()` function incorrectly calculates total assets by double counting underlying tokens and missing claimable balances, leading to inaccurate profit/loss reporting.

### Vulnerability Details

In `_harvestAndReport()`, the total assets calculation is:

```solidity
_totalAssets = unexchanged + asset.balanceOf(address(this)) + underlyingBalance;
```

Two key issues:

1. `underlyingBalance` should not be included since underlying tokens (WETH) are immediately swapped to asset (alETH) in `claimAndSwap()`
    
2. The function misses `claimable` balance from the transmuter which represents exchanged tokens that can be claimed
    

The correct calculation should be:

```solidity
_totalAssets = unexchanged + asset.balanceOf(address(this)) + claimable;
```

### Impact

- Incorrect total assets reporting leads to wrong profit/loss calculations in the TokenizedStrategy's `report()` function
    
- This affects:
    
    - Performance fee calculations
        
    - Share price calculations
        
    - Profit unlocking mechanism
        
- Users may receive wrong share amounts when depositing/withdrawing
    
- Protocol fees may be calculated incorrectly
    

### Tools Used

- Manual code review
    
- Understanding of TokenizedStrategy's `report()` mechanism
    

### Recommendations

1. Remove `underlyingBalance` from total assets calculation
    
2. Add `claimable` balance to the total:
    

```solidity
function _harvestAndReport() internal override returns (uint256 _totalAssets) {
    uint256 claimable = transmuter.getClaimableBalance(address(this));
    uint256 unexchanged = transmuter.getUnexchangedBalance(address(this));
    
    _totalAssets = unexchanged + asset.balanceOf(address(this)) + claimable;
}
```