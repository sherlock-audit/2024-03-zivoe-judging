Faithful Sky Wallaby

medium

# OLC_ZVE use of asset balances rather than provided amounts can make adding liquidity difficult

## Summary

`OLC_ZVE` allows the DAO to provide liquidity to UniswapV2 pools through their router. The balances provided to `router.addLiquidity()` use the locker's `balanceOf`, rather than the amounts provided in the call by the DAO. This means if there are left over funds from previous calls or from a direct transfer (potentially griefing), then the call to `addLiquidity` can fail due to the `desired` and `minimum` amounts provided in the call to the router.

## Vulnerability Detail

The DAO calls `pushToLockerMulti` with assets and amounts of those assets to be transfered into the locker and utilised to add liquidity to UniswapV2.

[OCL_ZVE.sol#L192-L206](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L192-L206)
```solidity
        uint balPairAsset = IERC20(pairAsset).balanceOf(address(this));
        uint balZVE = IERC20(ZVE).balanceOf(address(this));
        IERC20(pairAsset).safeIncreaseAllowance(router, balPairAsset);
        IERC20(ZVE).safeIncreaseAllowance(router, balZVE);

        // Prevent volatility of greater than 10% in pool relative to amounts present.
        (uint256 depositedPairAsset, uint256 depositedZVE, uint256 minted) = IRouter_OCL_ZVE(router).addLiquidity(
            pairAsset, 
            ZVE, 
            balPairAsset,
            balZVE, 
            (balPairAsset * 9) / 10,
            (balZVE * 9) / 10, 
            address(this), block.timestamp + 14 days
        );
```
However the amounts passed during the call to `addLiquidity` utilise the locker's balance, rather than the amounts provided by the DAO during the call. This can lead to issue with adding liquidity to the pool, for example if the pool is imbalanced.

## Impact

Calls to `addLiquidity` can fail due to the `desired` and `minimum` amounts provided in the call, as the amounts provided by the DAO are not the final values passed when adding liquidity. Meaning it may be difficult to add liquidity to imbalanced pools, where a lot of one token needs to be passed:

`Token1` to `Token2` ration in a pool: 100 to 1
DAO wants to add liquidity worth 10000 token 1 and 100 token 2.
Their addLiquidity call would have these parameters related to amounts:
`amountADesired = 10000`
`amountBDesired = 100`
`amountAMin = 9000`
`amountBMin = 90`

The add liquidity call would succeed.

If a malicious user wants to grief the DAO, they can send a donation of 15 `token2`, altering the passed amounts to addLiquidity:
`amountADesired = 10000`
`amountBDesired = 115`
`amountAMin = 9000`
`amountBMin = 103.5`

Now the call will fail as there is no way to add proportional funds to the LP at the correct rates. This can be costly for an attacker, however DAO transactions are subject to a timelock contract meanining that after one transaction is griefed there will be a period in which the DAO will have to make a new timelock transaction. Then the griefer can try to repeat a similar attack, meaning even though each grief attack is costly, the cost is only applied once every 12 hours (based on the timelock).

## Code Snippet

[OCL_ZVE.sol#L192-L206](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L192-L206)

## Tool used

Manual Review

## Recommendation

Add another parameter of `uint256[]` to the function. This 2nd `uint256[]` specifies what exact values should be passed to `IRouter_OCL_ZVE(router).addLiquidity`. This allows the DAO to transfer differing amount from the amounts that are passed to `addLiquidity`. 

An example of this proposed alteration:
Current locker balance: 1000 `token1` and 5 `token2`.
DAO can transfer 9000 `token1` and 100 `token2`
DAO can specify to pass 10000 `token1` and 100 `token2` to `addLiquidity`.
Any donations will be added to the lockers balance, but will not affect the call to `addLiquidity`. Leftover balance can be utilised in future locker calls.

```diff
    function pushToLockerMulti(
-        address[] calldata assets, uint256[] calldata amounts, bytes[] calldata data
+       address[] calldata assets, uint256[] calldata amounts, uint256[] calldata liquidityAmounts, bytes[] calldata data   
    ) external override onlyOwner nonReentrant {
        address ZVE = IZivoeGlobals_OCL_ZVE(GBL).ZVE();
        require(
            assets[0] == pairAsset && assets[1] == ZVE,
            "OCL_ZVE::pushToLockerMulti() assets[0] != pairAsset || assets[1] != ZVE"
        );

        for (uint256 i = 0; i < 2; i++) {
            require(amounts[i] >= 10 * 10**6, "OCL_ZVE::pushToLockerMulti() amounts[i] < 10 * 10**6");
            IERC20(assets[i]).safeTransferFrom(owner(), address(this), amounts[i]);
+          require(IERC20(assets[i]).balanceOf(address(this)) >= liquidityAmounts[i], "not enough balance")
        }

        if (nextYieldDistribution == 0) { nextYieldDistribution = block.timestamp + 30 days; }

        uint256 preBasis;
        if (basis != 0) { (preBasis,) = fetchBasis(); }

        // Router addLiquidity() endpoint.
-        uint balPairAsset = IERC20(pairAsset).balanceOf(address(this));
-       uint balZVE = IERC20(ZVE).balanceOf(address(this));
-        IERC20(pairAsset).safeIncreaseAllowance(router, balPairAsset);
-        IERC20(ZVE).safeIncreaseAllowance(router, balZVE);
+       IERC20(pairAsset).safeIncreaseAllowance(router, liquidityAmounts[0]);
+       IERC20(ZVE).safeIncreaseAllowance(router, liquidityAmounts[1]);

        // Prevent volatility of greater than 10% in pool relative to amounts present.
        (uint256 depositedPairAsset, uint256 depositedZVE, uint256 minted) = IRouter_OCL_ZVE(router).addLiquidity(
            pairAsset, 
            ZVE, 
-            balPairAsset,
-            balZVE, 
-            (balPairAsset * 9) / 10,
-            (balZVE * 9) / 10, 
+            liquidityAmounts[0],
+            liquidityAmounts[1],
+            (liquidityAmounts[0] * 9) / 10,
+            (liquidityAmounts[1] * 9) / 10,
            address(this), block.timestamp + 14 days
        );
```