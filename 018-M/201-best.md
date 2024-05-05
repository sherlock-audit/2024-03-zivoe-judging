Rural Sandstone Toad

medium

# EMA Data Point From Unlock Is Discarded

## Summary

The `ema` calculation fails to incorporate the `emaJTT` and `emaSTT` amounts which are initally set in `unlock`.

## Vulnerability Details

Let's carefully consider the EMA calculation. Firstly during the unlock, the `emaSTT` and `emaJTT `are set to the current values. This acts as our first data point. Since this is the only data point, this doesn't have any averaging which is why `MATH.ema()` is not called.

Now, after unlocking consider when `distributeYield` is called for the first time after unlocking. `emaSTT` and `emaJTT` should incorporate the first data point (which was recorded during unlock) with the new `totalSupply` of `STT` and `JTT`.

Now let's show why the contract will actually ignore the `emaSTT`/JTT set during unlock during the first yield distribution:

The `distributionCounter` is `0` when `distributeYield` is called, but due to the line `distributionCounter += 1;`, `1` is passed as the 3rd parameter to the `ema` formula in `emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter));`

When `N == 1` in the `ema` formula, the function will return only `eV`, which is only the latest data point. It completely ignores `bV` which is the data point during the unlock:

```solidity
function ema(uint256 bV, uint256 cV, uint256 N) external pure returns (uint256 eV) {

    assert(N != 0);

    uint256 M = (WAD * 2).floorDiv(N + 1); //When N = 1, M = WAD

    eV = ((M * cV) + (WAD - M) * bV).floorDiv(WAD); //Substituting M = WAD, eV = cV

}
```

## Impact

Incorrect EMA calculation which leads to incorrect yield distribution

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L213-L310

## Tool used

Manual Review

## Recommendation

The `distributionCounter` should actually be `2` rather than 1 during the first pass into the `MATH.ema()` formula. However then the varibale would not really correspond to the number of times `distribution` was called, so it might need to be renamed.
