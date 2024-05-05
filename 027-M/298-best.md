Cool Oily Seal

high

# ZivoeYDL::distributeYield yield distribution is flash-loan manipulatable

## Summary

`ZivoeYDL::distributeYield` is used to "distributes available yield within this contract to appropriate entities" but it relies on the tokens `totalSupply()` which can be manipulable through a flashloan.

## Vulnerability Detail

`ZivoeYDL::distributeYield` relies on `stZVE().totalSupply()` to distribute protocol earnings and residual earnings:

[ZivoeYDL.sol#L241-L310](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeYDL.sol#L241-L310)
```solidity
        // Distribute protocol earnings.
...
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                uint256 splitBIPS = (
>>                  IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                ) / (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                );
...
        // Distribute residual earnings.
...
                else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                    uint256 splitBIPS = (
>>                      IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                    ) / (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                        IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                    );
...
```

This can be abused by an attacker by buying then staking a very large amount of `ZVE` right before calling `ZivoeYDL::distributeYield` (a flashloan can be used) in order to game the system and collect a lot more distributed yields than he should be entitled to.

### Scenario

1. Attacker buys and stakes a very large amount of `ZVE` through a flashloan
2. Attacker calls `ZivoeYDL::distributeYield`
3. Attacker collects a very large amount of distributed yields
4. Attacker withdraws and sells back his `ZVE` tokens effectively stealing undeserved yields

## Impact

A user can systematically claim a big chunk of the rewards reserved to `stZVE` and `vestZVE` in `ZivoeYDL` by using a flash loan

## Code Snippet

- https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeYDL.sol#L241-L310

## Tool used

Manual Review

## Recommendation

Use a minimal time locking for stZVE such as it would not be possible to stake and unstake all in one block, or use past (1 block in the past) total supply to claim rewards