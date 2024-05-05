Clumsy Cobalt Lion

medium

# Time calculation issues with exponential decay

## Summary
[OCE_ZVE](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCE/OCE_ZVE.sol) is a locker that distributes rewards based on a exponential decay model. When a distribution starts, the whole balance of the contract should be unclaimable. As time moves forward, more and more of it should be unlocked for distributing. The calculations are made using the `lastDistribution` variable which is initially set in the constructor and then changed on each distribution. This is problematic because the "idle" periods where no distribution is happening will be considered as passed time when a real distribution starts.
## Vulnerability Detail
In the [constructor](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCE/OCE_ZVE.sol#L73C1-L73C44), the lastDistribution variable is set to `block.timestamp`. 
```solidity
        lastDistribution = block.timestamp;
```

When [forwardEmissions()](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCE/OCE_ZVE.sol#L131-L135) gets called, the calculation `block.timestamp - lastDistribution` will return a large value because the timer has started at the time of deployment.
```solidity
    function forwardEmissions() external nonReentrant {
        uint zveBalance = IERC20(IZivoeGlobals_OCE_ZVE(GBL).ZVE()).balanceOf(address(this));
        _forwardEmissions(zveBalance - decay(zveBalance, block.timestamp - lastDistribution));
        lastDistribution = block.timestamp;
    }
```
As we can see in the [Figma file](https://www.figma.com/file/qjuQ0uGQl9QD7KeBwyf73d/Zivoe-Visualization?type=whiteboard&node-id=0-1), the OCE locker will be deployed at `Phase One` and funding will start after ITO ends, which is at least 30 days.

This results  in a wrong calculation and instead of decaying, a big amount of the rewards can be forwarded as soon as the distribution starts.

The issue persists for further distributions also. If distribution 1 ends on 1 January and the Zivoe team decides to start distribution 2 on 1 July, the rewards for 6 months will be claimable from the very beginning. Clearing the timestamp before a distribution starts is not an option because it requires at least `100e18` assets to be forwarded.
```solidity
        require(amount >= 100 ether, "OCE_ZVE::_forwardEmissions amount < 100 ether");
```

## Impact
Instead of decaying, a big part of the rewards is claimable from the beginning.

## Code Snippet
PoC for [Test_OCE_ZVE](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-testing/src/TESTS_Lockers/Test_OCE_ZVE.sol)
```solidity
    function test_OCE_timer() public {
        hevm.warp(block.timestamp + 365 days);
        deal(address(ZVE), address(OCE_ZVE_Live), 20000e18);
        OCE_ZVE_Live.forwardEmissions();  
    }
```


## Tool used
Foundry

## Recommendation
Add a guarded function that start the distribution by updating the timestamp.
```solidity
function startDistribution() external {
         require(
            _msgSender() == IZivoeGlobals_OCE_ZVE(GBL).TLC(), 
            "OCE_ZVE::startDistribution() _msgSender() != IZivoeGlobals_OCE_ZVE(GBL).TLC()"
        );
        lastDistribution = block.timestamp;
}
```