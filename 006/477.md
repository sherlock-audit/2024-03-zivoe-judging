Flaky Burlap Boa

high

# Protocol unable to get extra Rewards in OCY_Convex_C

## Summary

Convex would wrap `rewardToken` for pools with IDs 151+, but the counting logic in `OCY_Convex_C.sol` makes it impossible for zivoe to forward yield.

## Vulnerability Detail

In `OCY_Convex_C.sol `, A convex pool with id `270` is used: 

    /// @dev Convex information.
    address public convexDeposit = 0xF403C135812408BFbE8713b5A23a04b3D48AAE31;
    address public convexPoolToken = 0x383E6b4437b59fff47B619CBA855CA29342A8559;
    address public convexRewards = 0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0;

    uint256 public convexPoolID = 270;

In the following logic, `rewardContract` is defaulted to the address of extraRewards. This assumption is fine for pools with PoolId < 150, but would not work for IDs 151+.

    /// @notice Claims rewards and forward them to the OCT_YDL.
    /// @param extra Flag for claiming extra rewards.
    function claimRewards(bool extra) public nonReentrant {
        IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();

        // Native Rewards (CRV, CVX)
        uint256 rewardsCRV = IERC20(CRV).balanceOf(address(this));
        uint256 rewardsCVX = IERC20(CVX).balanceOf(address(this));
        if (rewardsCRV > 0) { IERC20(CRV).safeTransfer(OCT_YDL, rewardsCRV); }
        if (rewardsCVX > 0) { IERC20(CVX).safeTransfer(OCT_YDL, rewardsCVX); }

        // Extra Rewards
        if (extra) {
            uint256 extraRewardsLength = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewardsLength();
            for (uint256 i = 0; i < extraRewardsLength; i++) {
                address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i); 
                //@Audit incorrect here!
                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }
            }
        }
    }


According to [convex doc](https://docs.convexfinance.com/convexfinanceintegration/baserewardpool#:~:text=Important%20for%20pools,wrappers/ConvexStakingWrapper.sol), 

> for pools with IDs 151+:
> VirtualBalanceRewardPool's rewardToken points to a wrapped version of the underlying token.  This Token implementation can be found here: https://github.com/convex-eth/platform/blob/main/contracts/contracts/StashTokenWrapper.sol

Just check `convexRewards` of pool 270: 

https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#readContract#F5

For index 0, it returns a [`VirtualBalanceRewardPool`](https://etherscan.io/address/0x22A0a706Aa423E2257e4217be2268e0374b9229f) with [rewardtoken](https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#readContract#F19) = 0x85D81Ee851D36423A5784CD3Cb6f1a1193Cb5978. This contract is a `StashTokenWrapper`, which is consistent with what the convex documentation says.

And, when `IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();` is triggered, reward tokens will be unwrapped and send to caller, so rewardAmount will always return 0, means such yield cannot be claimed for zivoe.

                address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i); 

                //@Audit incorrect here! `rewardToken` is a `StashTokenWrapper`, not the reward token!

                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }

## Impact

Users will lose extra rewards from convex pools with IDs 151+.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L210-L219
https://docs.convexfinance.com/convexfinanceintegration/baserewardpool

## Tool used

Manual Review

## Recommendation

Change the logic above to:

                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().token().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }
