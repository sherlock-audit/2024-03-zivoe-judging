Fresh White Barracuda

high

# `ZivoeTranches#rewardZVEJuniorDeposit` function miscalculates the reward when the ratio traverses lower/upper bound.

## Summary
`ZivoeTranches#rewardZVEJuniorDeposit` function miscalculates the reward when the ratio traverses lower/upper bound.
The same issue also exists in the `ZivoeTranches#rewardZVESeniorDeposit` function.

## Vulnerability Detail
`ZivoeTranches#rewardZVEJuniorDeposit` function is the following.
```solidity
    function rewardZVEJuniorDeposit(uint256 deposit) public view returns (uint256 reward) {

        (uint256 seniorSupp, uint256 juniorSupp) = IZivoeGlobals_ZivoeTranches(GBL).adjustedSupplies();

        uint256 avgRate;    // The avg ZVE per stablecoin deposit reward, used for reward calculation.

        uint256 diffRate = maxZVEPerJTTMint - minZVEPerJTTMint;

        uint256 startRatio = juniorSupp * BIPS / seniorSupp;
        uint256 finalRatio = (juniorSupp + deposit) * BIPS / seniorSupp;
213:    uint256 avgRatio = (startRatio + finalRatio) / 2;

        if (avgRatio <= lowerRatioIncentiveBIPS) {
216:        avgRate = maxZVEPerJTTMint;
        } else if (avgRatio >= upperRatioIncentiveBIPS) {
218:        avgRate = minZVEPerJTTMint;
        } else {
220:        avgRate = maxZVEPerJTTMint - diffRate * (avgRatio - lowerRatioIncentiveBIPS) / (upperRatioIncentiveBIPS - lowerRatioIncentiveBIPS);
        }

223:    reward = avgRate * deposit / 1 ether;

        // Reduce if ZVE balance < reward.
        if (IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this)) < reward) {
            reward = IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this));
        }
    }
```
Here, let us assume that `lowerRatioIncentiveBIPS = 1000`, `upperRatioIncentiveBIPS = 2500`, `minZVEPerJTTMint = 0`, `maxZVEPerJTTMint = 0.4 * 10 ** 18`, `seniorSupp = 10000`.

Let us consider the case of `juniorSupp = 0` where the ratio traverses the lower bound.

Example 1:
Assume that the depositor deposit `2000` at a time.
Then `avgRatio = 1000` holds in `L213`, thus `avgRate = maxZVEPerJTTMint = 0.4 * 10 ** 18` holds in `L216`.
Therefore `reward = 0.4 * deposit = 800` holds in `L223`.

Example 2:
Assume that the depositor deposit `1000` twice.
Then, since `avgRate = 500 < lowerRatioIncentiveBIPS` holds for the first deposit, `avgRate = 0.4 * 10 ** 18` holds in `L216`, thus `reward = 400` holds.
Since `avgRate = 1500 > lowerRatioIncentiveBIPS` holds for the second deposit, `avgRate = 0.3 * 10 ** 18` holds in `L220`, thus `reward = 300` holds.
Finally, the total sum of rewards for two deposits are `400 + 300 = 700`.

This shows that the reward of the case where all assets are deposited at a time is larger than the reward of the case where assets are divided and deposited twice. In this case, the protocol gets loss of funds.

Likewise, in the case where the ratio traverses the upper bound, the reward of one time deposit will be smaller than the reward of two times deposit and thus the depositor gets loss.

The same issue also exists in the `ZivoeTranches#rewardZVESeniorDeposit` function.

## Impact
When the ratio traverses the lower/upper bound in `ZivoeTranches#rewardZVEJuniorDeposit` and `ZivoeTranches#rewardZVESeniorDeposit` functions, the amount of reward will be larger/smaller than it should be. Thus the depositor or the protocol will get loss of funds.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L215-L221

## Tool used
Manual Review

## Recommendation
Modify the functions to calculate the reward dividing into two portion when the ratio traverses the lower/upper bound, which is similar to the case of Example 2.