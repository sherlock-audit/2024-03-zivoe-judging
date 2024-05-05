Able Cinnabar Tardigrade

medium

# Users are unable to receive any rewards if they are blacklisted by any of the reward tokens

## Summary

Users cannot get any rewards once they are only blacklisted by one `_rewardsTokens`.

## Vulnerability Detail

When claiming rewards, users must claim all rewards for all `_rewardTokens`. If a user is blacklisted by any one of the `_rewardTokens` (such as *USDT*), he cannot receive rewards for other `rewardTokens` because the `safeTransfer` will always revert the `getRewards` function.

```solidity
    /// @notice Claim rewards for all possible _rewardTokens.
    function getRewards() public updateReward(_msgSender()) {
        for (uint256 i = 0; i < rewardTokens.length; i++) { _getRewardAt(i); }
    }
    
    /// @notice Claim rewards for a specific _rewardToken.
    /// @param index The index to claim, corresponds to a given index of rewardToken[].
    function _getRewardAt(uint256 index) internal nonReentrant {
        address _rewardsToken = rewardTokens[index];
        uint256 reward = rewards[_msgSender()][_rewardsToken];
        if (reward > 0) {
            rewards[_msgSender()][_rewardsToken] = 0;
            IERC20(_rewardsToken).safeTransfer(_msgSender(), reward);
            emit RewardDistributed(_msgSender(), _rewardsToken, reward);
        }
    }
```


## Impact

Users cannot get any rewards once they are blacklisted by any one of the `_rewardsTokens`.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L281-L295

## Tool used

Manual Review

## Recommendation

Allow users to get rewards via specified rewardTokens.