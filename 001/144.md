Clumsy Cobalt Lion

medium

# `OCL` locker uses `block.timestamp` to calculate AMM deadline

## Summary
[`OCL_ZVE`](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L205) uses `block.timestamp + 14 days` as a deadline parameter when adding and removing liquidity. This is equivalent to having a disabled deadline feature, as `block.timestamp` is the timestamp at which the tx executes.

## Vulnerability Detail
Pushing to and pulling from the OCL locker adds and removes liquidity from a given pool. The code tries to set a deadline of the transaction to 2 weeks, meaning that transactions older than 2 weeks will be reverted. However, it does so by using `block.timestamp` which is ineffective. 

Transactions sent to the locker basically have no deadline and can be executed at any point in the future. The transaction may stay for a long time in the mempool due to network activity or be even abused by a MEV bot. Then the locker will be forced to execute a liqudity addition/removal even if its terms are unfavorable.
## Impact
Medium, since it requires external conditions to be met.

## Code Snippet
```solidity
        (uint256 depositedPairAsset, uint256 depositedZVE, uint256 minted) = IRouter_OCL_ZVE(router).addLiquidity(
            pairAsset, 
            ZVE, 
            balPairAsset,
            balZVE, 
            (balPairAsset * 9) / 10,
            (balZVE * 9) / 10, 
            address(this), block.timestamp + 14 days
        );
```

```solidity
            (uint256 claimedPairAsset, uint256 claimedZVE) = IRouter_OCL_ZVE(router).removeLiquidity(
                pairAsset, ZVE, preBalLPToken, 
                amountAMin, amountBMin, address(this), block.timestamp + 14 days
            );
```

## Tool used

Manual Review

## Recommendation
Instead of using `block.timestmap` encode the deadline in the `data` variable.
```diff
-        (uint amountAMin, uint amountBMin) = abi.decode(data, (uint, uint));
+       (uint amountAMin, uint amountBMin, uint deadline) = abi.decode(data, (uint, uint, uint));
```