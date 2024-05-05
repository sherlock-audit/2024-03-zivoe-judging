Uneven Inky Sealion

medium

# Unbalanced and imprecise stZVE and vestZVE token distribution occurs inside the ZivoeRewards contract due to the precision loss

## Summary
Less accurate stZVE and vestZVE tokens are calculated and distributed to the ZivoeRewards contract due to the priority of division over multiplication.

## Vulnerability Detail
Solidity rounds down the result of an integer division, and because of that, it is always recommended to multiply before 
dividing to avoid that precision loss. In the case of a prior division over multiplication, the final result may face serious precision loss
as the first answer would face truncated precision and then multiplied to another integer.

The problem arises in the ZivoeYDL's `distributeYield()` part. This function is responsible for distributing available yield. 
If we look deeply at this function, we can see how the allocated stZVE and vestZVE are calculated for the situation where the recipient is equal to `IZivoeGlobals_YDL(GBL).stZVE()`:

```Solidity
    function distributeYield() external nonReentrant {
        ...
            // Calculation of the protocol earnings
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                uint256 splitBIPS = (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                ) / (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                );
                uint stZVEAllocation = _protocol[i] * splitBIPS / BIPS;
                uint vestZVEAllocation = _protocol[i] * (BIPS - splitBIPS) / BIPS;
                IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).stZVE(), stZVEAllocation);
                IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).vestZVE(),vestZVEAllocation);
                IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).stZVE()).depositReward(distributedAsset, stZVEAllocation);
                IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).vestZVE()).depositReward(distributedAsset, vestZVEAllocation);

        ...

            // Calculation of the residual earnings
                else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                    uint256 splitBIPS = (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                    ) / (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                        IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                    );
                    uint stZVEAllocation = _residual[i] * splitBIPS / BIPS;
                    uint vestZVEAllocation = _residual[i] * (BIPS - splitBIPS) / BIPS;
                    IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).stZVE(), stZVEAllocation);
                    IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).vestZVE(), vestZVEAllocation);
                    IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).stZVE()).depositReward(distributedAsset, stZVEAllocation);
                    IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).vestZVE()).depositReward(distributedAsset, vestZVEAllocation);

        ...
    }
```

As it is illustrated, for the case where the recipient is `IZivoeGlobals_YDL(GBL).stZVE()`, the variables `stZVEAllocation`, and `vestZVEAllocation` are distributed to the mentioned recipient (for both
situations of protocol earnings or residual earnings).

These variables are calculated using the preceding formula: (just considering the protocol earning part that is similar to the residual earning part)

allocated stZVE formula:

$$ stZVE = \frac{protocol \ earnings \times splitBIPS}{BIPS} $$


allocated vestZVE formula:

$$ vestZVE = \frac{protocol \ earnings \times (BIPS - splitBIPS)}{BIPS} $$

At its heart, the `splitBIPS` is calculated as the division of stZVE total supply over the sum of stZVE and vestZVE total supplies:

$$ splitBIPS = \frac{stZVE \ totalSupply}{stZVE \ totalSupply + vestZVE \ totalSupply} $$

we can see in the actual implementation, there is a [hidden division before multiplication](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L250-L257) in the calculation of the `stZVEAllocation`, and `vestZVEAllocation` that rounds down the whole expression. This is bad as the precision loss can be significant, which leads to the pool calculating less accurate `stZVE`, and `vestZVE` than actual.

At the Proof of Concept part, we can check this behavior precisely.

You can run this code to see the difference between the results:

```Solidity
    function test_precissionLoss() public {

        uint stZVEtotalSupply = IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply();
        uint vestZVEtotalSupply = IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply();

        uint256 splitBIPS = (stZVEtotalSupply * BIPS) / 
        (
            stZVEtotalSupply + 
            vestZVEtotalSupply
        );
        uint stZVEAllocation = _protocol * splitBIPS / BIPS;
        uint vestZVEAllocation = _protocol * (BIPS - splitBIPS) / BIPS;

        uint stZVEAllocation_accurate = (_protocol * stZVEtotalSupply * BIPS) / (BIPS * (stZVEtotalSupply + vestZVEtotalSupply));
        uint vestZVEAllocation_accurate = (_protocol * vestZVEtotalSupply * BIPS) / (BIPS * (stZVEtotalSupply + vestZVEtotalSupply));
        
        console.log("Current Implementation of allocated stZVE  ", stZVEAllocation);
        console.log("Current Implementation of allocated vestZVE", vestZVEAllocation);

        console.log("Accurate Implementation of allocated stZVE  ", stZVEAllocation_accurate);
        console.log("Accurate Implementation of allocated vestZVE", vestZVEAllocation_accurate);
    }
```

The result would be: 

(for these sample but real variables: 

`stZVEtotalSupply = 6.62423125 ether`, 

`vestZVEAllocation = 4.5621235125 ether`,

`_protocol = 1563214500000`
)

```Solidity

     Current Implementation of allocated stZVE  : 925579305450
     Current Implementation of allocated vestZVE: 637635194550

     Accurate Implementation of allocated stZVE  : 925689785565
     Accurate Implementation of allocated vestZVE: 637524714434 
```
Thus, we can see that the actual implementation produces **less** allocated stZVE amount and **more** allocated vestZVE amount than the precise method.
The error becomes significant especially when the protocol earning becomes bigger.

## Impact
Precision loss inside the `distributeYield()` function leads to unbalanced stZVE and vestZVE token distributions and fund loss 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L250-L257
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L288-L298

## Tool used

Manual Review

## Recommendation
Consider modifying the allocated tokens calculation to prevent such precision loss and prioritize multiplication over division:

```diff
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
-                uint256 splitBIPS = (
-                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
-                ) / (
-                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
-                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
-                );
-                uint stZVEAllocation = _protocol[i] * splitBIPS / BIPS;
-                uint vestZVEAllocation = _protocol[i] * (BIPS - splitBIPS) / BIPS;
+                uint stZVEAllocation = _protocol[i] * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS) /
+                    (BIPS * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
+                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()));
+                uint vestZVEAllocation = _protocol[i] * (IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply() * BIPS) /
+                    (BIPS * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
+                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()));
```