Future Cherry Dog

medium

# Senior investor can not deposit in ZivoeTranches because of a DivisionError

## Summary
Senior investors get DivisionError and can't deposit when the supply of junior and senior are in default preventing the system to became helthy again.


## Vulnerability Detail
Look at `ZivoeTranches::depositSenior`

```javascript
    function depositSenior(uint256 amount, address asset) public notPaused nonReentrant {

        // ...

        //@audit this throws a DivisionError, reverts the transaction and doesn't allow Senior investor to deposit.
        uint256 incentives = rewardZVESeniorDeposit(convertedAmount);

        // ...
    }
```
When a Senior investor try to deposit it will calculate the incentives using the `ZivoeTranches::rewardZVESeniorDeposit` but it will throw a DivionError
that will prevent the system to became helthy again, see at `ZivoeTranches::rewardZVESeniorDeposit`

```javascript
    function rewardZVESeniorDeposit(uint256 deposit) public view returns (uint256 reward) {

        (uint256 seniorSupp, uint256 juniorSupp) = IZivoeGlobals_ZivoeTranches(GBL).adjustedSupplies();

        uint256 avgRate; 

        uint256 diffRate = maxZVEPerJTTMint - minZVEPerJTTMint;
        // @audit if seniorSupp == 0 -> throws a DivisionError
        uint256 startRatio = juniorSupp * BIPS / seniorSupp;
        uint256 finalRatio = juniorSupp * BIPS / (seniorSupp + deposit);
        uint256 avgRatio = (startRatio + finalRatio) / 2;
```

When the seniorSupp is 0, that means junior supply and senior suppply are in default, a divisionError will raise and will be imposibble to calculate rewards and allow Senior investors to deposit.


## Impact
Senior Investors can not deposit to make the system healthy again.

## Code Snippet
Paste this PoC to Test_ZivoeTranches.sol

```javascript
    function test_ZivoeTranches_poc_depositeSenior_div_zero() public {

        simulateITO(0 ether, 0 ether, 100_000_000 * USD, 100_000_000 * USD);

        (uint256 seniorSupp, uint256 juniorSupp) = GBL.adjustedSupplies();
        console2.log("Supply after ITO: seniorSupp %d. juniorSupp %d", seniorSupp, juniorSupp);
        
        hevm.startPrank(address(roy));
        zvl.try_updateIsLocker(address(GBL), address(roy), true);
        // There is a default of all the suppply.
        uint256 totalDefault = juniorSupp + seniorSupp;
        GBL.increaseDefaults(totalDefault);
        (uint256 seniorSuppAfterDefault, uint256 juniorSuppAfterDefault) = GBL.adjustedSupplies();
        console2.log("Supply after default: %d junior %d", seniorSuppAfterDefault, juniorSuppAfterDefault);
        
        hevm.startPrank(address(jim));
        mint("DAI", address(jim), 100 ether);
        jim.try_approveToken(address(DAI), address(ZVT), 100 ether);
        // Junior can't add funds because is closed so can't take the system out of this idle state.
        hevm.expectRevert("ZivoeTranches::depositJunior() !isJuniorOpen(amount, asset)");
        ZVT.depositJunior(100 ether, address(DAI));

        hevm.startPrank(address(sam));
        mint("DAI", address(sam), 50 ether);
        sam.try_approveToken(address(DAI), address(ZVT), 50 ether);
        // Seniors get divide by zero error and can't deposit so can't make the system healthy again
        hevm.expectRevert(stdError.divisionError);
        ZVT.depositSenior(50 ether, address(DAI));

    }
```

## Tool used

Manual Review

## Recommendation

In the `rewardZVESeniorDeposit` function when the seniorSupp == 0, set the avgRate = maxZVEPerJTTMint;
