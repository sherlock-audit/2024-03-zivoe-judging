Petite Velvet Duck

high

# distributeYield() calls earningsTrancheuse() with outdated emaSTT & emaJTT while calculating senior & junior tranche yield distributions

## Summary
The [distributeYield()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L229-L239) function internally calls `earningsTrancheuse()` on [L232](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L232) which calculates & returns the `_seniorTranche` & `_juniorTranche` yield distribution. However, this call results in `earningsTrancheuse()` using [outdated values of](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L461-L462) `emaSTT` & `emaJTT` on L461-L462 as these variables are updated only on [L238-L239](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L238-L239) _after_ the call to `earningsTrancheuse()` is concluded.

## Code Snippet
```js
  File: src/ZivoeYDL.sol

  213:              function distributeYield() external nonReentrant {
  214:                  require(unlocked, "ZivoeYDL::distributeYield() !unlocked"); 
  215:                  require(
  216:                      block.timestamp >= lastDistribution + daysBetweenDistributions * 86400, 
  217:                      "ZivoeYDL::distributeYield() block.timestamp < lastDistribution + daysBetweenDistributions * 86400"
  218:                  );
  219:          
  220:                  // Calculate protocol earnings.
  221:                  uint256 earnings = IERC20(distributedAsset).balanceOf(address(this));
  222:                  uint256 protocolEarnings = protocolEarningsRateBIPS * earnings / BIPS;
  223:                  uint256 postFeeYield = earnings.floorSub(protocolEarnings);
  224:          
  225:                  // Update timeline.
  226:                  distributionCounter += 1;
  227:                  lastDistribution = block.timestamp;
  228:          
  229:                  // Calculate yield distribution (trancheuse = "slicer" in French).
  230:                  (
  231:                      uint256[] memory _protocol, uint256 _seniorTranche, uint256 _juniorTranche, uint256[] memory _residual
  232: @--->            ) = earningsTrancheuse(protocolEarnings, postFeeYield); 
  233:          
  234:                  emit YieldDistributed(_protocol, _seniorTranche, _juniorTranche, _residual);
  235:                  
  236:                  // Update ema-based supply values. 
  237:                  (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
  238: @--->            emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter));
  239: @--->            emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter));
                        ...
                        ...
```

and

```js
  File: src/ZivoeYDL.sol

  447:              function earningsTrancheuse(uint256 yP, uint256 yD) public view returns (
  448:                  uint256[] memory protocol, uint256 senior, uint256 junior, uint256[] memory residual
  449:              ) {
  450:                  protocol = new uint256[](protocolRecipients.recipients.length);
  451:                  residual = new uint256[](residualRecipients.recipients.length);
  452:                  
  453:                  // Accounting for protocol earnings.
  454:                  for (uint256 i = 0; i < protocolRecipients.recipients.length; i++) {
  455:                      protocol[i] = protocolRecipients.proportion[i] * yP / BIPS;
  456:                  }
  457:          
  458:                  // Accounting for senior and junior earnings.
  459:                  uint256 _seniorProportion = MATH.seniorProportion(
  460:                      IZivoeGlobals_YDL(GBL).standardize(yD, distributedAsset),
  461: @--->                MATH.yieldTarget(emaSTT, emaJTT, targetAPYBIPS, targetRatioBIPS, daysBetweenDistributions),
  462: @--->                emaSTT, emaJTT, targetAPYBIPS, targetRatioBIPS, daysBetweenDistributions
  463:                  );
  464:                  senior = (yD * _seniorProportion) / RAY;
  465:                  junior = (yD * MATH.juniorProportion(emaSTT, emaJTT, _seniorProportion, targetRatioBIPS)) / RAY; 
  466:                                                                                                                    
  467:                  // Handle accounting for residual earnings.
  468:                  yD = yD.floorSub(senior + junior);
  469:                  for (uint256 i = 0; i < residualRecipients.recipients.length; i++) {
  470:                      residual[i] = residualRecipients.proportion[i] * yD / BIPS;
  471:                  }
  472:              }
```

## Vulnerability Detail
As the [docs explain](https://docs.zivoe.com/user-docs/yield-distribution#liquidity-providers:~:text=Returns%20are%20calculated%20using%20the%20adjusted%20supply%20of%20tranche%20tokens%2C%20with%20an%20Exponential%20Moving%20Average%20(EMA)%20playing%20a%20significant%20role), the EMA plays a significant role in yield distribution calculations.

> Returns are calculated using the adjusted supply of tranche tokens, with an Exponential Moving Average (EMA) playing a significant role. 

> The EMA smoothens the change in the supply of tranche tokens over a look-back period of 2.5 months.

The EMA is used to calculate the:
- yield target via a call to [MATH.yieldTarget()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeMath.sol#L113-L121) which expects "_ema-based supply of zSTT & zJTT_" as its params.
- senior tranche yield proportion via a call to [MATH.seniorProportion()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeMath.sol#L59-L70) which again expects ema-based `yT, eSTT & eJTT`.

Both the above mentioned calls happen inside `earningsTrancheuse()` [here](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L461-L462). The issue is that these global variables `emaSTT and emaJTT` are still outdated and correspond to the ones belonging to the `lastDistribution` timestamp instead of current updated ones. These values are updated only after the call to `earningsTrancheuse()` has concluded [here](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L238-L239).

## Impact
Imagine that in the last 30 days, JTT supply has reduced due to defaulted loans. As a result, the target yield $yT_{real}$ would come down too (i.e. $yT_{real}$ < $yT$ where $yT$ is the protocol calculated incorrect target yield) and the junior tranche ditributable yield will be smaller than before. However if there is excess yield, then the junior tranche would receive more than their fair share since last 30 days have not been considered by the protocol calculations. If the quantum of defaulted loans is high (and hence $yT_{real}$ has dipped significantly), the impact is not only limited to the junior tranche, but then also effects the senior tranche. <br>

On the flip side an increase in the adjusted supplies of STT and JTT in the last 30 days will have a impact on the smoothened emaSTT and emaJTT, making the values go higher. As a result, target yield $yT_{real}$ would go up too. Thus, it could happen that `yD` is less than $yT_{real}$ but the protocol is oblivious to that due to usage of outdated values. This would result in the protcol using [seniorProportionBase()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeMath.sol#L74) to calculate the senior's yield instead of [seniorProportionShortfall()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeMath.sol#L72). 
<br>

Finally, since at each of these function calls an outdated eSTT is passed as a param, the value returned would be more/less than what it should be, thus causing gain/loss of yield for both the tranches ([juniorProportion()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeMath.sol#L74) calculation has the same problem).
<br>

**_Note:_** For the very first distribution i.e. when `distributionCounter = 1`, the values used are from 60 days ago instead of 30 days [as can be seen inside `unlock()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L328-L331), further increasing the margin of error. The 60 day gap has initially been provided by the protocol to allow for more time to bring in initial yield.

## Tool used
Manual Review

## Recommendation
Update the ema values **_before_** calling `earningsTrancheuse()`:
```diff
+       // Update ema-based supply values.
+       (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
+       emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter));
+       emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter));

        // Calculate yield distribution (trancheuse = "slicer" in French).
        (
            uint256[] memory _protocol, uint256 _seniorTranche, uint256 _juniorTranche, uint256[] memory _residual
        ) = earningsTrancheuse(protocolEarnings, postFeeYield); 

        emit YieldDistributed(_protocol, _seniorTranche, _juniorTranche, _residual);
        
-       // Update ema-based supply values.
-       (uint256 aSTT, uint256 aJTT) = IZivoeGlobals_YDL(GBL).adjustedSupplies();
-       emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter));
-       emaJTT = MATH.ema(emaJTT, aJTT, retrospectiveDistributions.min(distributionCounter));
```