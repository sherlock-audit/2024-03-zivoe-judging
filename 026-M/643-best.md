Brisk Latte Wallaby

high

# A user can let the last payment of his loan `default` to avoid paying `lateFee and interestOwed`

## Summary

A user can let the last payment of his loan `default` to avoid paying `lateFee and interestOwed`

## Vulnerability Detail

In the `OCC_Modular.resolveDefault` function, if the `loans[id].principalOwed` is totally repaid for a `defaulted loan` the state of the loan is updated to the `LoanState.Resolved`. And after that the `underwriter` can call the  `OCC_Modular.markRepaid` function to change the state back to `loans[id].state = LoanState.Repaid`.

So when the loan is in the `LoanState.Repaid` state it is considered to be in the fully settled state.

But the issue is when the `OCC_Modular.resolveDefault` is called only the `loans[id].principalOwed` is repaid back to the `DAO` and it does not account for the `lateFee or the interestOwed`. This is in contrast to how the loan is repaid in the `OCC_Modular.makePayment` and `OCC_Modular.callLoan` functions. Because when these two functions are called to repay a `loanPayment` it accounts for the `lateFee` and the `interestOwed` and the `payer` has to `repay` the `principal + lateFee + intereseOwed`.

## Impact

As a result of the above vulnerability, a user can let his `loan` default (last payment of his loan) and  avoid paying the `lateFee` rather than calling the `OCC_Modular.makePayment` or `OCC_Modular.callLoan` function where he has to pay the `lateFee` if the payment is delayed. This will result in lost funds to the `IZivoeGlobals_OCC(GBL).YDL()` or `OCT_YDL` contracts accordingly. (Because usually late fees and owed interest are paid to these contracts).

## Code Snippet

```solidity
        uint256 paymentAmount;

        if (amount >= loans[id].principalOwed) {
            paymentAmount = loans[id].principalOwed;
            loans[id].principalOwed = 0;
            loans[id].state = LoanState.Resolved;
        }
        else {
            paymentAmount = amount;
            loans[id].principalOwed -= paymentAmount;
        }

        emit DefaultResolved(id, paymentAmount, _msgSender(), loans[id].state == LoanState.Resolved);

        IERC20(stablecoin).safeTransferFrom(_msgSender(), owner(), paymentAmount);
        IZivoeGlobals_OCC(GBL).decreaseDefaults(IZivoeGlobals_OCC(GBL).standardize(paymentAmount, stablecoin));
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L688-L703

```solidity
        (uint256 principalOwed, uint256 interestOwed, uint256 lateFee,) = amountOwed(id);

        emit PaymentMade(
            id, _msgSender(), principalOwed + interestOwed + lateFee, principalOwed,
            interestOwed, lateFee, loans[id].paymentDueBy + loans[id].paymentInterval
        );

        // Transfer interest + lateFee to YDL if in same format, otherwise keep here for 1INCH forwarding.
        if (stablecoin == IZivoeYDL_OCC(IZivoeGlobals_OCC(GBL).YDL()).distributedAsset()) {
            IERC20(stablecoin).safeTransferFrom(_msgSender(), IZivoeGlobals_OCC(GBL).YDL(), interestOwed + lateFee);
        }
        else {
            IERC20(stablecoin).safeTransferFrom(_msgSender(), OCT_YDL, interestOwed + lateFee);
        }
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L578-L591

```solidity
        uint256 principalOwed = loans[id].principalOwed;
        (, uint256 interestOwed, uint256 lateFee,) = amountOwed(id);

        emit LoanCalled(id, principalOwed + interestOwed + lateFee, principalOwed, interestOwed, lateFee);

        // Transfer interest to YDL if in same format, otherwise keep here for 1INCH forwarding.
        if (stablecoin == IZivoeYDL_OCC(IZivoeGlobals_OCC(GBL).YDL()).distributedAsset()) {
            IERC20(stablecoin).safeTransferFrom(_msgSender(), IZivoeGlobals_OCC(GBL).YDL(), interestOwed + lateFee);
        }
        else {
            IERC20(stablecoin).safeTransferFrom(_msgSender(), OCT_YDL, interestOwed + lateFee);
        }
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L499-L510

## Tool used
Manual Review and VSCode

## Recommendation

Hence it is recommended to compute the `lateFee` and the `interestOwed` for the `defaulted loan payment` when the `OCC_Modular.resolveDefault` function is called and charge that payment from the payer (msg.sender) and transfer it to the `IZivoeGlobals_OCC(GBL).YDL()` contract or the `OCT_YDL` contract accordingly.
