Dancing Violet Gorilla

medium

# Title: Inadequate Allowance Handling in convertAndForward Function of `OCT_DAO` & `OCT_YDL`.

## Summary
The `OCT_DAO:convertAndForward `and `OCT_YDL:convertAndForward `function suffers from inadequate handling of token allowances for the 1inch router. Although allowances are set correctly before converting assets, they are not reset afterward. This can lead to failed transactions if the 1inch router does not utilize the entire allowance due to slippage or other factors.

## Vulnerability Detail

the issue arises from the lack of allowance reset after interacting with the 1inch router. If the router does not utilize the entire allowance specified due to slippage or other reasons, the allowance will remain unchanged, potentially causing the assertion to fail and the transaction to revert.

## Impact

The impact of this vulnerability is that transactions may fail due to incorrect allowance management. This could result in inefficiencies in liquidity provision, loss of gas fees because due to asset statement it consume all the gas and won't return anything and convertandTransfer will never be able to transfer asset to DAO.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCT/OCT_DAO.sol#L87
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCT/OCT_YDL.sol#L97
```javascript
assert(IERC20(asset).allowance(address(this), router1INCH_V5) == 0);
```

## Tool used

Manual Review

## Recommendation

Reset Allowances After Use: After interacting with the Uniswap router and completing the liquidity provision, reset the allowances for the pair assets and ZVE tokens to zero to ensure they are not left with excessive allowances.
