Rural Sandstone Toad

high

# Tranche Depositors Are Punished For Defaults that Happened in the past

## Summary

`AdjustedSupplies` adjusts the `zJTT` and `zSTT` amount to account for defaults. However deposits are not adjusted based on `adjustedSupplies()` which means that stakers get less than `$1` of adjusted tranche tokens per deposit and losses that should go to the stakers during the default are incorrectly socialized to future stakers.

## Vulnerability Detail

The `adjustedSupplies` is applied to interest collection and eventual withdrawal. However it is not applied to deposits. That means that instead of getting $1 worth of zJTT, the user gets less than $1, which means that after a single default, nobody would want to deposit. Instead, the default should be socialised to current depositors, and the deposit should get $1 of backing. 

**Example**

There is $100 in junior tranche. Then $90 default.

The user deposits $100 into the junior tranche in exchange for 100 zJTT. However calling `adjustedSupplies`, you get 110 / 200 * 100 = $55, so you get less than $1 worth of zJJT per $1 you deposit.

## Impact

After any default happens, depositors get less than $1 worth of `zJTT` or `zSTT` and are punished for defaults that happened in the past.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeGlobals.sol#L126-L136

## Tool used

Manual Review

## Recommendation

In case of a large default, stakers should get more than 1 zJTT per $ deposited to adjust for the loss for defaults. However, care must be taken because they `decreaseDefaults` would increase the worth of all existing `zJTT` when it should only be credited to those who originally were punished for the default. 
