Fun Lilac Bee

medium

# `ZivoeTranches.depositSenior()` not protected agains zSTT balance = 0

## Summary

Contract: `ZivoeTranches`

Functions: `depositSenior()` => `rewardZVESeniorDeposit()`

The function `depositSenior()` is not protected against the value of zSTT total supply = 0 but there is 2 scenarios where this could happen:

1. A unsuccessful ITO phase, with 0 deposits.
2. Users decided to burn all the current zSTT deposits.

Remember to never allow the option to a variable or parameter to be 0 if it is is in the hands of the users.


## Vulnerability Detail

The problem is located in `rewardZVESeniorDeposit()`:

```solidity
uint256 startRatio = juniorSupp * BIPS / seniorSupp;
uint256 finalRatio = (juniorSupp + deposit) * BIPS / seniorSupp;
```

Check the test:

```solidity
function test_depositSeniorBlocked() external unlockTranches {
        assertEq(zSTT.totalSupply(), 0);
        vm.startPrank(seniorLPMaryJane);
        DAI.approve(address(ZVT), 100 ether);
        vm.expectRevert();
        ZVT.depositJunior(100 ether, address(DAI));
        vm.stopPrank();
    }
```

```terminal
➜  zivoe-core-foundry git:(main) ✗ forge test --mt test_depositSeniorBlocked
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Phase3_CORE.sol:Phase3_CORE
[PASS] test_depositSeniorBlocked() (gas: 124773)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.05ms (177.92µs CPU time)

Ran 1 test suite in 141.24ms (10.05ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
➜  zivoe-core-foundry git:(main) ✗

```

## Impact

This is a HIGH RISK vulnerability as the protocol get blocked without options of deposits as junior tranches need that the senior tranches works correctly.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L211

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L212

## Tool used

Manual Review

## Recommendation

There is 2 ways to proceed:

1. An initial, and protected, zSTT mint().
3. protect the `rewardZVESeniorDeposit()` against zSTT = 0 with a require.

