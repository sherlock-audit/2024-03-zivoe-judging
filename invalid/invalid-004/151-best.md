Cheery Lemon Nuthatch

medium

# Vesting Could End in a Life Time

## Summary
wrongly converting days to vest value, which is already expected to be in days time to days, overstates the vesting period to a far greater value than intended, possibly locking users' ZVE for a lifetime
## Vulnerability Detail
[ZivoeRewardsVesting::createVestingSchedule](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L381-L425) is intended to be called by the ZVL or the ITO contract to create a vesting for a user.
In [createVestingSchedule](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L381-L425) we have this check:
```solidity
        require(
            daysToVest <= 1800 days,
            "ZivoeRewardsVesting::createVestingSchedule() daysToVest > 1800 days"
        );
```
The code seems to intend to restrict vesting time to a maximum of 1800 days(~5 years). But when we look at where `daysToVest` is used down the logic, it is multiplied by `1 days`, to convert a normal `uint` to days time
```solidity
        vestingScheduleOf[account].cliff =
            block.timestamp +
            daysToCliff *
            1 days;
        vestingScheduleOf[account].end = block.timestamp + daysToVest * 1 days;
        vestingScheduleOf[account].totalVesting = amountToVest;
        vestingScheduleOf[account].vestingPerSecond =
            amountToVest /
            (daysToVest * 1 days);
```
The inconsistency here is that,  the requirement check, assumes the input `daysToVest` is in days, but the computation assumes it isn't, and thus multiplies it by 1 day. 

Let me Illustrate:

+ If the ZVL passed in a `daysToVest` in days, i.e. 2 days, which will be 172800 seconds, the actual days to vest used will be 172800 days(~473 years)

Since this breaks the invariant of restricting vesting days to a maximum of 1800 days but  instead allows up to a max of 155520000 days(~ 426082 years), and also since this could have users ZVE locked for the rest of their lives, without a way to reset this values or a way to create another vesting for the user, I believe this issue should qualify as a medium
## Impact
ZVE could be locked for a lifetime
## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L381-L425
## Tool used

Manual Review

## Recommendation
Adjust the requirement to:
```solidity
        require(
            daysToVest <= 1800,
            "ZivoeRewardsVesting::createVestingSchedule() daysToVest > 1800"
        );
```