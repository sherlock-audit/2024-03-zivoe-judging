Clumsy Cobalt Lion

medium

# Revoking vesting schedule does not subtract user votes correctly

## Summary
[ZivoeVestingRewards.revokeVestingSchedule()](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L429-L467) should reduce the voting power of the user with the withdrawable amount plus the revoked amount. However, it reduces it only by the withdrawable amount.

## Vulnerability Detail
When called, `revokeVestingSchedule()` fetches the withdrawable amount by the user at that moment.
```solidity
        uint256 amount = amountWithdrawable(account);
```

The revoke logic is executed and the user's checkpoint value is decreased by `amount`.
```solidity
        _writeCheckpoint(_checkpoints[account], _subtract, amount);
```

The code ignores the amount that's being revoked and the user keeps more voting power than he has to.
Imagine the following:
`totalVested = 1000`
`withdrawable = 0`

If the schedule gets revoked, the user's checkpoint value will not be decreased at all because there is nothing to be withdrawn. The user can later use their voting power to vote on governance proposals.

In fact, `amountWithdrawable(account)` being close to 0 has a very high likelihood because:
   - the user can frontrun the transaction and withdraw the tokens they are entitled to
   -  it's highly likely that a vesting schedule will be removed shortly after creating it.

However, even if `amountWithdrawable()` is not equal to 0, the user would still be left with more voting power.
## Impact
Users keep voting power that must have been taken away.

## Code Snippet
POC to be run in [Test_ZivoeRewardsVesting.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewardsVesting.sol)
```solidity
    function test_revoking_leaves_votes() public {
       assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(moe), 
            0, 
            360,
            6000 ether, 
            true
        ));
        // Vesting succeeded
        assertEq(vestZVE.balanceOf(address(moe)), 6000 ether);

        hevm.roll(block.number + 1);
     
        // User votes have increased
        assertEq(vestZVE.getPastVotes(address(moe), block.number - 1), 6000 ether);

        assert(zvl.try_revokeVestingSchedule(address(vestZVE), address(moe)));
        // Revoking succeeded
        assertEq(vestZVE.balanceOf(address(moe)), 0);

        hevm.roll(block.number + 1);
        // User votes have not been decreased at all
        assertEq(vestZVE.getPastVotes(address(moe), block.number - 1), 6000 ether);
    }
```

## Tool used

Foundry

## Recommendation
Subtract the correct amount from the checkpoint's value
```diff
-        _writeCheckpoint(_checkpoints[account], _subtract, amount);
+       _writeCheckpoint(_checkpoints[account], _subtract, vestingAmount - vestingScheduleOf[account].totalWithdrawn + amount)
```

