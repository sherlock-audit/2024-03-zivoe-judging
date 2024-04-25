Able Cinnabar Tardigrade

medium

# Anyone could call `depositReward` with zero reward to extend the period finish time

## Summary

Anyone could extend the reward finish time, potentially resulting in users receiving fewer rewards than expected within the same time period.

## Vulnerability Detail

The function `depositReward` can be called by anyone, even with zero rewards, allowing it to be exploited to extend the reward finish time at little cost. 
This could result in loss of rewards; for instance, if there are 10 DAI rewards within a 10-day period, a malicious user could extend the finish time on *day 5*, extending the finish time to the 15th day. Participants would only receive 7.5 DAI by the 10th day.

```solidity
    function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
        IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);

        // Update vesting accounting for reward (if existing rewards being distributed, increase proportionally).
        if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
            rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
        } else {
            uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
            rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
        }

        rewardData[_rewardsToken].lastUpdateTime = block.timestamp;
        rewardData[_rewardsToken].periodFinish = block.timestamp.add(rewardData[_rewardsToken].rewardsDuration);
        emit RewardDeposited(_rewardsToken, reward, _msgSender());
    }
```

### POC
Add the test to `zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol` and run it with `forge test --match-test test_ZivoeRewards_deposit_zero --rpc-url <RPC_URL_MAINNET>`

```diff
diff --git a/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol b/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
index f5353b6..870a531 100644
--- a/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
+++ b/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
@@ -685,6 +685,33 @@ contract Test_ZivoeRewards is Utility {

     }

+    function test_ZivoeRewards_deposit_zero() public {
+
+        depositReward_DAI(address(stZVE), 1);
+
+        (
+            uint256 rewardsDuration,
+            uint256 _prePeriodFinish,
+            uint256 _preRewardRate,
+            uint256 lastUpdateTime,
+            uint256 rewardPerTokenStored
+        ) = stZVE.rewardData(DAI);
+        console.log("period finish ", _prePeriodFinish);
+
+        vm.warp(block.timestamp + 1 days);
+
+        depositReward_DAI(address(stZVE), 0);
+
+        (,
+            uint256 _afterPeriodFinish,
+            ,
+            ,
+        ) = stZVE.rewardData(DAI);
+        console.log("period finish ", _afterPeriodFinish);
+        //  extend the Finish 1 day
+        assertEq(_afterPeriodFinish - _prePeriodFinish, 1 days);
+    }
+
     function test_ZivoeRewards_getRewards_works(uint96 random) public {

         uint256 deposit = uint256(random) + 100 ether; // Minimum 100 DAI deposit.
```

## Impact

Anyonce could extend the reward finish time and the users may receive less rewards than expected during the same time period.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243

## Tool used

Foundry

## Recommendation
Only specific users are allowed to call function `depositReward`