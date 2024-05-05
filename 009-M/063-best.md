Fierce Slate Gorilla

medium

# No slippage for incentive.

## Summary
When a user calls `depositJunior` or `depositSenior` functions in `ZivoeTranches`, he will deposit a stablecoin and receive `incentives` and will be minted tranche token 1:1 with his deposit. However `incentives` are dynamic and he can receive less than wanted as there is no slippage control.

## Vulnerability Detail
The `incentives` are calculated in `rewardZVEJuniorDeposit` and `rewardZVESeniorDeposit`. As can be seen, the return value is dynamic as there are multiple calculations and validation.

## Impact
User receiving less `incentives`.

## Code Snippet
```solidity
    function rewardZVEJuniorDeposit(uint256 deposit) public view returns (uint256 reward) {

        (uint256 seniorSupp, uint256 juniorSupp) = IZivoeGlobals_ZivoeTranches(GBL).adjustedSupplies();

        uint256 avgRate;    // The avg ZVE per stablecoin deposit reward, used for reward calculation.

        uint256 diffRate = maxZVEPerJTTMint - minZVEPerJTTMint;

        uint256 startRatio = juniorSupp * BIPS / seniorSupp;
        uint256 finalRatio = (juniorSupp + deposit) * BIPS / seniorSupp;
        uint256 avgRatio = (startRatio + finalRatio) / 2;

        if (avgRatio <= lowerRatioIncentiveBIPS) {
            avgRate = maxZVEPerJTTMint;
        } else if (avgRatio >= upperRatioIncentiveBIPS) {
            avgRate = minZVEPerJTTMint;
        } else {
            avgRate = maxZVEPerJTTMint - diffRate * (avgRatio - lowerRatioIncentiveBIPS) / (upperRatioIncentiveBIPS - lowerRatioIncentiveBIPS);
        }

        reward = avgRate * deposit / 1 ether;

        // Reduce if ZVE balance < reward.
        if (IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this)) < reward) {
            reward = IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this));
        }
    }
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L203-L229

```solidity
    function rewardZVESeniorDeposit(uint256 deposit) public view returns (uint256 reward) {

        (uint256 seniorSupp, uint256 juniorSupp) = IZivoeGlobals_ZivoeTranches(GBL).adjustedSupplies();

        uint256 avgRate;    // The avg ZVE per stablecoin deposit reward, used for reward calculation.

        uint256 diffRate = maxZVEPerJTTMint - minZVEPerJTTMint;

        uint256 startRatio = juniorSupp * BIPS / seniorSupp;
        uint256 finalRatio = juniorSupp * BIPS / (seniorSupp + deposit);
        uint256 avgRatio = (startRatio + finalRatio) / 2;

        if (avgRatio <= lowerRatioIncentiveBIPS) {
            avgRate = minZVEPerJTTMint;
        } else if (avgRatio >= upperRatioIncentiveBIPS) {
            avgRate = maxZVEPerJTTMint;
        } else {
            avgRate = minZVEPerJTTMint + diffRate * (avgRatio - lowerRatioIncentiveBIPS) / (upperRatioIncentiveBIPS - lowerRatioIncentiveBIPS);
        }

        reward = avgRate * deposit / 1 ether;

        // Reduce if ZVE balance < reward.
        if (IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this)) < reward) {
            reward = IERC20(IZivoeGlobals_ZivoeTranches(GBL).ZVE()).balanceOf(address(this));
        }
    }
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L236-L262

## Tool used
Manual Review

## Recommendation
Add another parameter `minIncetives` to `depositJunior` and `depositSenior` functions. Then check if `incentives >= minIncetives`.