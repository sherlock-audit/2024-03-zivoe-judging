Clumsy Cobalt Lion

medium

# No slippage protection in `OCL_ZVE.forwardYield()`

## Summary
When forwarding yield, [OCL_ZVE](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L316-L318) removes liquidity from a given Uni/Sushi pool. However, it has set `amount0Min` and `amount1Min` both to 0 making itself vulnerable to sandwich attacks and price fluctuations.

## Vulnerability Detail
The contract can execute a very unfavorable liquidity removal either by natural price fluctuations or by becoming a victim of a sandwich attack. An attacker, for example, can frontrun the call to forward yield and force the contract to receive a very small amount of `pairAsset` which will result in distributing a miniscule amount of yield to the tranche token stakers.

## Impact
Distributing little to no yield; executing an unfavorable trade.
## Code Snippet
```solidity
        (uint256 claimedPairAsset, uint256 claimedZVE) = IRouter_OCL_ZVE(router).removeLiquidity(
            pairAsset, ZVE, lpBurnable, 0, 0, address(this), block.timestamp + 14 days
        );
```

## Tool used

Manual Review

## Recommendation
A possible solution may be to use the `basis` and calculate slippage protection parameters out of it.
