Elegant Silver Scorpion

high

# Unhandled situation upon combining loans leads to loss of funds for the protocol

## Summary
Whenever a borrower is late on his loan repayment, he is supposed to pay a `lateFee`. However, if his loans are approved for combining, then he can be late on his loan repayment, owe money to the protocol but avoid paying them leading to a loss of funds for the protocol.

## Vulnerability Detail
Every active loan in `OCC_Modular` has a `paymentDueBy` property holding the `block.timestamp` when a `borrower` is supposed to make his repayment. If he is delayed on his repayment, he is supposed to pay a `lateFee` to the protocol calculated in the following way:
```solidity
function amountOwed(uint256 id) public view returns (
        uint256 principal, uint256 interest, uint256 lateFee, uint256 total
    ) {
        // 0 == Bullet.
        if (loans[id].paymentSchedule == 0) {
            if (loans[id].paymentsRemaining == 1) { principal = loans[id].principalOwed; }
        }
        // 1 == Amortization (only two options, use else here).
        else { principal = loans[id].principalOwed / loans[id].paymentsRemaining; }

        // Add late fee if past loans[id].paymentDueBy.
        if (block.timestamp > loans[id].paymentDueBy && loans[id].state == LoanState.Active) {
            lateFee = loans[id].principalOwed * (block.timestamp - loans[id].paymentDueBy) *
                loans[id].APRLateFee / (86400 * 365 * BIPS);
        }
        interest = loans[id].principalOwed * loans[id].paymentInterval * loans[id].APR / (86400 * 365 * BIPS);
        total = principal + interest + lateFee;
    }
```
The contract also allows for the combining of loans. First, an `underwriter` has to approve a combination using the `OCC_Modular::approveCombine()`. Then, the `borrower` has 3 days (72 hours) to approve the combination before it expires using the `OCC_Modular::applyCombine()` function.
```solidity
    function approveCombine(
        uint256[] calldata loanIDs,
        uint256 APRLateFee,
        uint256 term,
        uint256 paymentInterval,
        uint256 gracePeriod,
        int8 paymentSchedule
    ) external isUnderwriter {
        require(term > 0, "OCC_Modular::approveCombine() term == 0");
        require(
            paymentInterval == 86400 * 7 || paymentInterval == 86400 * 14 || paymentInterval == 86400 * 28 || 
            paymentInterval == 86400 * 91 || paymentInterval == 86400 * 364, 
            "OCC_Modular::approveCombine() invalid paymentInterval value, try: 86400 * (7 || 14 || 28 || 91 || 364)"
        );
        require(
            loanIDs.length > 1 && paymentSchedule <= 1 && gracePeriod >= 7 days, 
            "OCC_Modular::approveCombine() loanIDs.length <= 1 || paymentSchedule > 1 || gracePeriod < 7 days"
        );

        emit CombineApproved(
            combineCounter, loanIDs, APRLateFee, term, paymentInterval, gracePeriod, block.timestamp + 72 hours, paymentSchedule
        );
        
        combinations[combineCounter] = Combine(
            loanIDs, APRLateFee, term, paymentInterval, gracePeriod, block.timestamp + 72 hours, paymentSchedule, true
        );

        combineCounter += 1;
    }
```
```solidity
    function applyCombine(uint256 id) external {
        require(combinations[id].valid, "OCC_Modular::applyCombine() !combinations[id].valid");
        require(
            block.timestamp < combinations[id].expires, 
            "OCC_Modular::applyCombine() block.timestamp >= combinations[id].expires"
        );

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

        APR = APR / notional;

        uint256 term = combinations[id].term;
        uint256 APRLateFee = combinations[id].APRLateFee;
        uint256 paymentInterval = combinations[id].paymentInterval;
        uint256 gracePeriod = combinations[id].gracePeriod;
        int8 paymentSchedule = combinations[id].paymentSchedule;
        
        // "Friday" Payment Standardization, minimum 7-day lead-time
        // block.timestamp - block.timestamp % 7 days + 9 days + paymentInterval
        emit CombineLoanCreated(
            _msgSender(),  // borrower
            loanCounter,  // loanID
            notional,  // principalOwed
            APR,  // APR
            APRLateFee,  // APRLateFee
            block.timestamp - block.timestamp % 7 days + 9 days + paymentInterval,  // paymentDueBy
            term,  // term
            paymentInterval,  // paymentInterval
            gracePeriod,  // gracePeriod
            paymentSchedule  // paymentSchedule
        );
        loans[loanCounter] = Loan(
            _msgSender(), notional, APR, APRLateFee, block.timestamp - block.timestamp % 7 days + 9 days + paymentInterval, 
            term, term, paymentInterval, block.timestamp - 1 days, gracePeriod, paymentSchedule, LoanState.Active
        );
        loanCounter += 1;
    }
```
Upon making a repayment the usual way using the `OCC_Modular::makePayment()`, then the `lateFees` are taken into account. However, the `OCC_Modular::applyCombine()` function does not take into account the `lateFees` a `borrower` might owe to the protocol leading to a loss of funds for the protocol in the event of the `borrower` being late on his repayment.
## Impact
Protocol will not get all of the funds that they are required and expecting to get and borrowers will be able to delay their repayments without any consequences as long as their loans are approved for a combination.

## Proof Of Concept
Note that this POC showcases a very realistic and not very harsh case. An `underwriter` approves a combination a day before the `paymentDueBy` for the loans when in theory, he could even approve a combination after the `paymentDueBy` causing greater losses. Also, the POC is combining just 2 loans but the losses will be more severe if there are 3, 4, 5 or even more loans approved for combining. Also, the borrowed amount could be greater than the one in the POC causing greater loss.
Paste the following function into `Test_OCC_Modular.sol`:
```solidity
       function testLossOfLateFees() public {
        address underwriter = address(roy);
        address borrower = makeAddr('borrower');

        uint256 gracePeriod = 7 * 1 days;

         hevm.startPrank(underwriter); // Create loan offers to the borrower (10,000$ if stablecoin == DAI)
         OCC_Modular_DAI.createOffer(borrower, 10000e18, 1000, 1000, 24, 86400 * 7, gracePeriod, 1);
         OCC_Modular_DAI.createOffer(borrower, 10000e18, 1000, 1000, 24, 86400 * 7, gracePeriod, 1);
         hevm.stopPrank();

         hevm.startPrank(borrower); // Accept the loan offers
         OCC_Modular_DAI.acceptOffer(0);
         OCC_Modular_DAI.acceptOffer(1);
         hevm.stopPrank();

        uint256[] memory loanIds = new uint256[](2);
        loanIds[0] = 0;
        loanIds[1] = 1;

        (, , uint256[10] memory infoFirstLoan) = OCC_Modular_DAI.loanInfo(0);
        (, , uint256[10] memory infoSecondLoan) = OCC_Modular_DAI.loanInfo(1);
        
        uint256 paymentDueByFirstLoan = infoFirstLoan[3];
        uint256 paymentDueBySecondLoan = infoSecondLoan[3];
        
        assertEq(paymentDueByFirstLoan, paymentDueBySecondLoan);

        vm.warp(paymentDueByFirstLoan - 1 days); // Warp to before the payment is due for the first loan, a very realistic time for an underwriter to approve a combination
        hevm.prank(underwriter);
        OCC_Modular_DAI.approveCombine(loanIds, 1000, 24, 86400 * 7, gracePeriod, 1);

        vm.warp(block.timestamp + 3 days - 1); // Warp to before the combination expires

        (, , uint256 lateFeeFirstLoan, ) =  OCC_Modular_DAI.amountOwed(0);
        (, , uint256 lateFeeSecondLoan, ) =  OCC_Modular_DAI.amountOwed(1);

        console.log(lateFeeFirstLoan);
        assertEq(lateFeeFirstLoan, lateFeeSecondLoan); // 5479420345002536783

        hevm.prank(borrower);
        OCC_Modular_DAI.applyCombine(0);

        (, , uint256 lateFeeCombinedLoan, ) =  OCC_Modular_DAI.amountOwed(2);
        assertEq(lateFeeCombinedLoan, 0);

        uint256 lostFees = (lateFeeFirstLoan + lateFeeSecondLoan) - lateFeeCombinedLoan;
        console.log(lostFees);
        assertEq(lostFees, 5479420345002536783 * 2); // 10958840690005073566
    }
```
## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L742-L808
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L440-L457

## Tool used

Manual Review

## Recommendation
Make sure to handle a case where a borrower might be late on his repayment when combining his loans in a way you see fit (e.g., adding the late fees to the principalOwed, directly getting them from the user's balance upon calling of `applyCombine()`).