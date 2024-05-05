Faithful Sky Wallaby

medium

# ZivoeYDL::distributeYield() will revert if protocolRecipients recipients length is smaller than residualRecipients

## Summary

`ZivoeYDL::distributeYield` incorrectly accesses `_protocol[i]` instead of `_residual[i]`, and there are no checks anywhere within the code that ensures that both arrays are of the same length. Meaning that it is likely that this will cause `distributeYield()` to revert in the case where `protocolRecipients` length is less than `residualRecipients`.

## Vulnerability Detail

`ZivoeYDL::updateRecipients` allows the timelock contract to update the `protocolRecipients` or `residualRecipients` variables:

[ZivoeYDL::updateRecipients](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L392-L416)
```solidity
    function updateRecipients(address[] memory recipients, uint256[] memory proportions, bool protocol) external {
        require(_msgSender() == IZivoeGlobals_YDL(GBL).TLC(), "ZivoeYDL::updateRecipients() _msgSender() != TLC()");
        require(
            recipients.length == proportions.length && recipients.length > 0, 
            "ZivoeYDL::updateRecipients() recipients.length != proportions.length || recipients.length == 0"
        );
        require(unlocked, "ZivoeYDL::updateRecipients() !unlocked");
... SKIP!...
        if (protocol) {
            emit UpdatedProtocolRecipients(recipients, proportions);
            protocolRecipients = Recipients(recipients, proportions);
        }
        else {
            emit UpdatedResidualRecipients(recipients, proportions);
            residualRecipients = Recipients(recipients, proportions);
        }
    }
```
It's important to note this function only sets one of those 2 variables at a time, and the length checks on `recipients` and `proportions` is to ensure that the `Recipients()` struct inputs have the same length, NOT to ensure that `protocolRecipients` and `residualRecipients` are the same length. 

`ZivoeYDL::distributeYield` calculates protocol earnings and distributes the yield to both the `protocolRecipients` and `residualRecipients`. It does so by looping through the recipients and the earnings:

[ZivoeYDL::distributeYield](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L213-L286)
```solidity
    function distributeYield() external nonReentrant {
        require(unlocked, "ZivoeYDL::distributeYield() !unlocked"); 
        require(
            block.timestamp >= lastDistribution + daysBetweenDistributions * 86400, 
            "ZivoeYDL::distributeYield() block.timestamp < lastDistribution + daysBetweenDistributions * 86400"
        );

        // Calculate protocol earnings.
        uint256 earnings = IERC20(distributedAsset).balanceOf(address(this));
        uint256 protocolEarnings = protocolEarningsRateBIPS * earnings / BIPS;
        uint256 postFeeYield = earnings.floorSub(protocolEarnings);

        // Update timeline.
        distributionCounter += 1;
        lastDistribution = block.timestamp;

        // Calculate yield distribution (trancheuse = "slicer" in French).
        (
            uint256[] memory _protocol, uint256 _seniorTranche, uint256 _juniorTranche, uint256[] memory _residual
        ) = earningsTrancheuse(protocolEarnings, postFeeYield); 

... SKIP!...

        // Distribute protocol earnings.
        for (uint256 i = 0; i < protocolRecipients.recipients.length; i++) {
            address _recipient = protocolRecipients.recipients[i];
            if (_recipient == IZivoeGlobals_YDL(GBL).stSTT() ||_recipient == IZivoeGlobals_YDL(GBL).stJTT()) {
                IERC20(distributedAsset).safeIncreaseAllowance(_recipient, _protocol[i]);
                IZivoeRewards_YDL(_recipient).depositReward(distributedAsset, _protocol[i]);
                emit YieldDistributedSingle(distributedAsset, _recipient, _protocol[i]);
            }
 
... SKIP!...

        // Distribute residual earnings.
        for (uint256 i = 0; i < residualRecipients.recipients.length; i++) {
            if (_residual[i] > 0) {
                address _recipient = residualRecipients.recipients[i];
                if (_recipient == IZivoeGlobals_YDL(GBL).stSTT() ||_recipient == IZivoeGlobals_YDL(GBL).stJTT()) {
                    IERC20(distributedAsset).safeIncreaseAllowance(_recipient, _residual[i]);
                    IZivoeRewards_YDL(_recipient).depositReward(distributedAsset, _residual[i]);
                    emit YieldDistributedSingle(distributedAsset, _recipient, _protocol[i]);
                }
```
The function loops over `protocolRecipients` and `residualRecipients` seperately and within each loop indexed positions are accessed in  `_protocol[]` and ` _residual[]`, these 2 arrays are initated as follows:

[ZivoeYDL::earningsTrancheuse](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L447-L451)
```solidity
    function earningsTrancheuse(uint256 yP, uint256 yD) public view returns (
        uint256[] memory protocol, uint256 senior, uint256 junior, uint256[] memory residual
    ) {
        protocol = new uint256[](protocolRecipients.recipients.length);
        residual = new uint256[](residualRecipients.recipients.length);
```
Their lengths will depend on the lengths of `protocolRecipients` and `residualRecipients` respectively, meaning their lengths can differ.

Finally, the 2nd for loop in `distributeYield` itterates over `residualRecipients` and 
 incorrectly emits the `YieldDistributedSingle` event by accessing `_protocol[i]`, whilst it should be using `_residual[i]`. This will cause the function to revert due to an out of bound index access if:

`residualRecipients.recipients.length > protocolRecipients.recipients.length`
AND
atleast one of the recipients in `residualRecipients.recipients` is:
`IZivoeGlobals_YDL(GBL).stSTT() || IZivoeGlobals_YDL(GBL).stJTT()`

which are the reward contracts:
[ZivoeYDL.sol#L20-L24](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L20-L24)
```solidity
    /// @notice Returns the address of the ZivoeRewards ($zSTT) contract.
    function stSTT() external view returns (address);

    /// @notice Returns the address of the ZivoeRewards ($zJTT) contract.
    function stJTT() external view returns (address);
```
meaning yield will not be able to be successfully distributed and the function will always revert as long as this condition is met.

## Impact

Core functionality of `ZivoeYDL` will be unaccessible due to the function always reverting as long as the above condition is met, and this condition can easily be met through normal protocol use. 

This can be fixed by ensuring that both array are of the same length, however this means that appropriate fees cannot be distributed to the correct parties.

## Code Snippet

[ZivoeYDL.sol#L392-L416](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L392-L416)
[ZivoeYDL.sol#L213-L286](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L213-L286)
[ZivoeYDL.sol#L447-L451](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L447-L451)
[ZivoeYDL.sol#L20-L24](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L20-L24)

## Tool used

Manual Review

## Recommendation

Change the incorrect emit to access from `_residual[i]` rather than `_protocol[i]`:
[ZivoeYDL.sol#L286](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L286)
```diff
- emit YieldDistributedSingle(distributedAsset, _recipient, _protocol[i]);
+ emit YieldDistributedSingle(distributedAsset, _recipient, _residual[i]);
```