Fantastic Corduroy Shark

medium

# `basis` in OCL_ZVE will underflow causing DOS of pullFromLockerPartial

## Summary
In the `OCL_ZVE.sol` the basis variable is used to keep track of how much pairAsset can be redeemed with the current LP positions. However an inconsistency will cause it to underflow, therefore causing revert int the `pullFromLockerPartial` function.
## Vulnerability Detail
When calling `pushToLockerMulti`, the postBasis - preBasis are added to the current basis:
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L214
When calling `pullFromLockerPartial`, the preBasis - postBasis is subtracted from the stored basis. However this line could cause basis to underflow blocking the `pullFromLockerPartial` function.
Consider the following scenario.
1) The owner calls `pushToLockerMulti` pushing 50 pair asset tokens to the liquidity pool. The rate for this scenario will be 1lp token = 5 pairAssetTokens (this means that pairAssetBalance of the pool devided by poolTotalSupply equals 10). Meaning that the locker will gain 50/5 = 10 lp tokens setting basis to 50. 
2) Under certain market conditions after 15 days the 10lp tokens are now worth 100pairAsset tokens(1lp =10 AssetToken). 
3) The owner decides to call `pullFromLockerPartial`, trying to remove 3lp tokens. In this case:
preBasis = 10*10 = 100. 
postBasis = 3*10 = 30.
preBasis-postBasis = 100-3-=70 Which is greater than the stored basis=50. As a result, the pullFromLockerPartial will revert.

## Impact
The `pullFromLockerPartial` will revert until the forwardYield function is called to reset the basis which could take up to 30 days.
## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L279
## Tool used

Manual Review

## Recommendation
Forward yielf to the OCT_YDL and adjust the basis in the beginning of pullFromLockerPartial.
```diff
function pullFromLockerPartial(address asset, uint256 amount, bytes calldata data)
        external
        override
        onlyOwner
        nonReentrant
    {
+         (uint256 amountToForward, uint256 lpCurrent) = fetchBasis();
+       if (amountToForward > basis) _forwardYield(amountToForward, lpCurrent);
+       (basis,) = fetchBasis();
        address ZVE = IZivoeGlobals_OCL_ZVE(GBL).ZVE();
        address pair = IFactory_OCL_ZVE(factory).getPair(pairAsset, ZVE);

        (uint256 amountAMin, uint256 amountBMin) = abi.decode(data, (uint256, uint256));

        // "pair" represents the liquidity pool token (minted, burned).
        // "pairAsset" represents the stablecoin paired against $ZVE.
        if (asset == pair) {
            (uint256 preBasis,) = fetchBasis();
            IERC20(pair).safeIncreaseAllowance(router, amount);

            // Router removeLiquidity() endpoint.
            (uint256 claimedPairAsset, uint256 claimedZVE) = IRouter_OCL_ZVE(router).removeLiquidity(
                pairAsset, ZVE, amount, amountAMin, amountBMin, address(this), block.timestamp + 14 days
            );
            emit LiquidityTokensBurned(amount, claimedZVE, claimedPairAsset);
            assert(IERC20(pair).allowance(address(this), router) == 0);

            IERC20(pairAsset).safeTransfer(owner(), IERC20(pairAsset).balanceOf(address(this)));
            IERC20(ZVE).safeTransfer(owner(), IERC20(ZVE).balanceOf(address(this)));
            (uint256 postBasis,) = fetchBasis();
            require(postBasis < preBasis, "OCL_ZVE::pullFromLockerPartial() postBasis >= preBasis");
            basis -= preBasis - postBasis;
        } else {
            IERC20(asset).safeTransfer(owner(), amount);
        }
    }
```

