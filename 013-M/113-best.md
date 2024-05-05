Fierce Crepe Sloth

medium

# Rewards are calculated as distributed even if there are no stakers, locking the rewards forever

## Summary
The ``ZivoeRewards.sol`` and the ``ZivoeRewardsVesting.sol`` contracts are a fork of the Synthetix rewards distribution contract, with slight modifications. The contract logic keeps track of the total time passed, as well as the cumulative rate of rewards that have been generated for each token used for rewards, so that for every user, it just has to keep track of the last timestamp and last rate at the time of the last reward withdrawal, stake, or unstake in order to calculate the rewards owed since the user began staking. The code special-cases the scenario where there are no users, by not updating the cumulative rate when the _totalSupply is zero, but it does not include such a condition for the tracking of the timestamp. Because of this, even when there are no users staking, the accounting logic still thinks funds were being dispersed during that timeframe (because the starting timestamp is updated), which means the funds effectively are distributed to nobody, rather than being saved for when there is someone to receive them. And will be locked in the contract forever. One of the modifications is this line in the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function. 
```solidity
      IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);
```
Contrary to the Synthetix implementation, Zivo requires each time reward is deposited to the contact, the reward amount to be transferred in the same transaction. However if a reward is deposited but there are no stakers, the reward that should have been distributed for the period until the first user stakes, will be locked in the contract forever, and won't be able to be added to the rewards for the next reward period because of the above code snippet. 

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the ``AuditorTests.t.sol`` contract:

```solidity
    function test_LockedRewards() public {
        vm.startPrank(ZVL);
        zivoeToken.transfer(alice, 5_000_000e18);
        uint256 duration = 30 days;
        stZVE.addReward(address(mockUSDC), duration);
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        mockUSDC.mint(simulateYLD, 50_000e6); // this represent 50_000 USDC tokens to be distributed in the next 30 days
        mockUSDC.approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(address(mockUSDC), 50_000e6);
        skip(172_800); /// @notice two days pass, before anybody stakes in the contract
        vm.stopPrank();

        vm.startPrank(alice);
        console2.log("USDC balance of alice before staking: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract: ", mockUSDC.balanceOf(address(stZVE)));
        zivoeToken.approve(address(stZVE), type(uint256).max);
        stZVE.stake(5_000_000e18);
        skip(duration);
        stZVE.fullWithdraw();
        console2.log("USDC balance of alice after staking for 30 days: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period: ", mockUSDC.balanceOf(address(stZVE)));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized: ", mockUSDC.balanceOf(address(stZVE)) / 1e6);
        console2.log("zivoToken balance of alice after unstaking: ", zivoeToken.balanceOf(alice));
        vm.stopPrank();
    }
```

```solidity
Logs:
  USDC balance of alice before staking:  0
  USDC balance of stZVE contract:  50000000000
  USDC balance of alice after staking for 30 days:  46665000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period:  3335000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized:  3335
  zivoToken balance of alice after unstaking:  5000000000000000000000000
```
As can be seen from the above logs **3335 USDC tokens** will be locked forever. 

To run the test use: ``forge test -vvv --mt test_LockedRewards``
## Impact
If the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function is called prior to there being any users staking, the funds that should have gone to the first stakers will instead accrue to nobody, and be locked in the contract forever. 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243

## Tool used
Manual Review & Foundry

## Recommendation
Remove the ``safeTransfer()`` from the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function, and instead check if there are enough reward tokens already in the contract. Something similar to the [Synthetix implementation](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol#L113-L132).