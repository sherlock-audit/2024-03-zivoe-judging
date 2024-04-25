Able Cinnabar Tardigrade

medium

# The function `applyCombine` has been incorrectly implemented

## Summary

The `applyCombine` function guarantees that each combination can only be invoked once by checking the `combinations[id].valid` status. However, it erroneously sets `combinations[combineCounter].valid` to false while leaving `combinations[id].valid` unchanged, resulting in incorrect behavior.

## Vulnerability Detail

The `combinations[id].valid` is used to ensure that each combination can only be executed once. However, the `applyCombine` wrongly set the `combinations[combineCounter].valid` to false, which should set the `combinations[id].valid` to false.

```solidity
    function applyCombine(uint256 id) external {
    //@audit Check for the valid status of *id*
        require(combinations[id].valid, "OCC_Modular::applyCombine() !combinations[id].valid");
        require(
            block.timestamp < combinations[id].expires, 
            "OCC_Modular::applyCombine() block.timestamp >= combinations[id].expires"
        );
    //@audit BUT set the *combineCounter* to false
        combinations[combineCounter].valid = false;

        emit CombineApplied(
            _msgSender(),
            combinations[id].loans, 
            combinations[id].term,
            combinations[id].paymentInterval, 
            combinations[id].gracePeriod,
            combinations[id].paymentSchedule
        );

        uint256 notional;
        uint256 APR;
        
        for (uint256 i = 0; i < combinations[id].loans.length; i++) {
            uint256 loanID = combinations[id].loans[i];
            require(
                _msgSender() == loans[loanID].borrower, 
                "OCC_Modular::applyCombine() _msgSender() != loans[loanID].borrower"
            );
            require(
                loans[loanID].state == LoanState.Active, 
                "OCC_Modular::applyCombine() loans[loanID].state != LoanState.Active"
            );
            notional += loans[loanID].principalOwed;
            APR += loans[loanID].principalOwed * loans[loanID].APR;
            loans[loanID].principalOwed = 0;
            loans[loanID].paymentDueBy = 0;
            loans[loanID].paymentsRemaining = 0;
            loans[loanID].state = LoanState.Combined;
        }

        [...]
    }
```


## Impact

The `applyCombine` function guarantees that each combination can only be invoked once by checking the `combinations[id].valid` status. However, it erroneously sets `combinations[combineCounter].valid` to false while leaving `combinations[id].valid` unchanged, resulting in incorrect behavior.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L742-L749

## Tool used

Manual Review

## Recommendation

Set the `combinations[id].valid` to false rather than the `combinations[combineCounter].valid`.