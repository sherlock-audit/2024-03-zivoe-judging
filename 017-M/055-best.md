Elegant Silver Scorpion

medium

# Forwarding yield in `OCL_ZVE` is possible a lot more often than the enforced 30 days

## Summary
Calling `OCL_ZVE::forwardYield()` is supposed to only be possible every 30 days however in reality, it is possible to call the function a lot more often.

## Vulnerability Detail
The `require` statement in `OCL_ZVE::forwardYield()` enforces that the function should only be callable when `block.timestamp` is of larger value than the value of `nextYieldDistribution`
```solidity
    function forwardYield() external {
        if (IZivoeGlobals_OCL_ZVE(GBL).isKeeper(_msgSender())) {
            require(
                block.timestamp > nextYieldDistribution - 12 hours, 
                "OCL_ZVE::forwardYield() block.timestamp <= nextYieldDistribution - 12 hours"
            );
        }
        else {
            require(
                block.timestamp > nextYieldDistribution, 
                "OCL_ZVE::forwardYield() block.timestamp <= nextYieldDistribution"
            );
        }

        (uint256 amount, uint256 lp) = fetchBasis();
        if (amount > basis) { _forwardYield(amount, lp); }
        (basis,) = fetchBasis();
        nextYieldDistribution += 30 days;
    }
```
Then, at the end of the function, `nextYieldDistribution` is incremented by 30 days making the function supposedly not callable for another 30 days. However, there is a vulnerability that allows `OCL_ZVE::forwardYield()` to be called a lot more times.

Upon calling the `OCL_ZVE::pushToLockerMulti()`, the `nextYieldDistribution` gets set to `block.timestamp + 30 days` if its value is 0.
```solidity
if (nextYieldDistribution == 0) { nextYieldDistribution = block.timestamp + 30 days; }
```
If this is the way `nextYieldDistribution` gets its first value, then there will not be an issue. However, if `OCL_ZVE::forwardYield()` is called beforehand, then `nextYieldDistribution` will be set to `0 += 30 days` which equals `30 days` making its value a lot less than the value of `block.timestamp` resulting in people being able to call `OCL_ZVE::forwardYield()` at will all the way until `nextYieldDistribution` gets incremented all the way to `block.timestamp`.

Calling `OCL_ZVE::forwardYield()` before any other function is possible and requires just 1 simple circumstance to be a fact.

Imagine the following scenario:
1. `OCL_ZVE::forwardYield()` is called
2. The `if` statement passes as `block.timestamp` is larger than `nextYieldDistribution`, the value of which is the default value of 0
3. `OCL_ZVE::fetchBasis()` gets called
```solidity
function fetchBasis() public view returns (uint256 amount, uint256 lp) {
        address pool = IFactory_OCL_ZVE(factory).getPair(pairAsset, IZivoeGlobals_OCL_ZVE(GBL).ZVE());
        uint256 pairAssetBalance = IERC20(pairAsset).balanceOf(pool);
        uint256 poolTotalSupply = IERC20(pool).totalSupply();
        lp = IERC20(pool).balanceOf(address(this));
        amount = lp * pairAssetBalance / poolTotalSupply;
    } 
```
4. As long as there is a pool setup for `pairAsset` and `ZVE`, the vulnerability will take place
5. Pool gets the value of the pool address
6. The only thing which has to pass here is `poolTotalSupply` not being 0 as division by 0 is not possible
7. That successfully passes as there is already a setup pool for `pairAsset` and `ZVE`
8. Then, we get back to the `OCL_ZVE::forwardYield()` function
9. `if(amount > basis)` does not pass as both values are 0
10. On the last line, `nextYieldDistribution` gets set to `30 days` making the `OCL_ZVE::forwardYield()` function callable again and again

## Impact
`OCL_ZVE::forwardYield()` is callable over and over again even though it is only supposed to be called every 30 days

## Proof Of Concept
Paste the following test into `Test_OCL_ZVE.sol`:
```solidity
function testCanForwardYieldALot() public {
        address UNIV2_ROUTER = OCL_ZVE_UNIV2_DAI.router();
        deal(DAI, address(this), 10000);

        IERC20(DAI).safeApprove(UNIV2_ROUTER, 10000);
        IERC20(ZVE).safeApprove(UNIV2_ROUTER, 1001);

        IUniswapV2Router01(UNIV2_ROUTER).addLiquidity(address(DAI), address(ZVE), 10000, 1001, 0, 0, address(this), block.timestamp);

        uint256 count;
        while (block.timestamp > OCL_ZVE_UNIV2_DAI.nextYieldDistribution()) {
            OCL_ZVE_UNIV2_DAI.forwardYield();
            count++;
        }

        console.log(count);
    }
```

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L186
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L287-L305

## Tool used

Manual Review

## Recommendation
Do not allow `OCL_ZVE::forwardYield()` to be called whenever the value of `nextYieldDistribution` is equal to 0. 
Also, another option is to set `nextYieldDistribution` to `block.timestamp + 30 days` instead of just incrementing it by `30 days`.
