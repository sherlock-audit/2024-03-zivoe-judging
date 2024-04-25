Bubbly Rouge Porcupine

medium

# It is not possible to retrieve some reward tokens ever from `OCY_Convex_C` and `OCY_Convex_A`

## Summary
The issue cannot be low, but medium because the roles of external integration are restricted. So, the `rewardManager` role of `convex` contracts are not trusted. They have the power to add or remove the reward tokens from the convex reward contarcts. So if they remove any reward tokens before the `OCY_Convex_C` claims its rewards, then these rewards are wasted permanently on `OCY_Convex_C` and `OCY_Convex_A`.

Look at the simple recommendation that can solve this issue.
It is not possible to retrieve some tokens ever. It is due to the lack of feature to move tokens from locker to YDL / DAO.

## Vulnerability Detail

From https://github.com/sherlock-audit/2024-03-zivoe?tab=readme-ov-file#q-are-the-admins-of-the-protocols-your-contracts-integrate-with-if-any-trusted-or-restricted

```md
Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?

RESTRICTED
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L200-L220

```solidity

File: 2024-03-zivoe\zivoe-core-foundry\src\lockers\OCY\OCY_Convex_C.sol

200:     function claimRewards(bool extra) public nonReentrant {
201:         IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();
202:
203:         // Native Rewards (CRV, CVX)
204:         uint256 rewardsCRV = IERC20(CRV).balanceOf(address(this));
205:         uint256 rewardsCVX = IERC20(CVX).balanceOf(address(this));
206:         if (rewardsCRV > 0) { IERC20(CRV).safeTransfer(OCT_YDL, rewardsCRV); }
207:         if (rewardsCVX > 0) { IERC20(CVX).safeTransfer(OCT_YDL, rewardsCVX); }
208:
209:         // Extra Rewards
210:         if (extra) {
211:            
212:             uint256 extraRewardsLength = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewardsLength();
213:             for (uint256 i = 0; i < extraRewardsLength; i++) {
214:  >>>            address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i);
215:  >>>            uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(
216:                     address(this)
217:                 );
218:                 if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }
219:                
220:             }
221:         }
222:     }

```


**Issue path:**

1. The `rewardManager` which is a restricted external  role of `convexRewards`, will add/remove a new reward token and it will stream new reward tokens for may be 7 days or even 2 - 4 weeks. And after a while, that reward token can be removed by that manager.

scroll a bit after going to https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#code#L835

![image](https://github.com/sherlock-audit/2024-03-zivoe-ironsidesec/assets/162350329/ba1353dd-a7ac-4bd7-a084-8e4f5223d645)



2. But `claimRewards`  was not called in any of these times so reward is not claimed, or maybe the reward manager has called `clearExtraRewards` before this claimreward call, which will remove the reward token. So there's no way `OCY_Convex_C` can claim reward except someone will call `getReward` externally with `OCY_Convex_C` as input address, and reward pending tokens will be transfered.

3. Still, the tokens cannot be transferred to YDL because the token addresses are called from line 214 above, which can be manipulated by `rewardManager`. so effectively wasted rewards.


## Impact
Loss of reward tokens in 2 convex integration contracts A and C. So medium.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L200-L220

https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_A.sol#L246

## Tool used

Manual Review

## Recommendation

Add the below function to both `OCY_Convex_C` and `OCY_Convex_A` contracts. So if there's any extra reward token in these contracts, they can be transferred out.

```solidity
    function canPullMulti() public override pure returns (bool) { return true; }

```

   