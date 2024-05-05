Quaint Brick Troll

high

# Contract assumes all the stablecoins are always 1:1 with USD which is not the case incase of a depeg event.

## Summary
Because the contract assume stablecoins are always 1:1 with USD and directly uses the amount to mint zJTT/zSTT, an attacker can profit from it incase one of the stablecoin depegs. 
See past depeg events here: [USDT](https://cointelegraph.com/news/untethered-the-history-of-stablecoin-tether-and-how-it-has-lost-its-1-peg), [DAI , USDC](https://cointelegraph.com/news/circle-s-usdc-instability-causes-domino-effect-on-dai-usdd-stablecoins).

## Vulnerability Detail
Since the contract doesn't use oracles, in case one stablecoin depegs(say USDC) and goes down in value( < USD ), it could be utilized by an attacker to mint more tranche tokens(zJTT/zSTT) than he should and profit from it by redeeming to another stablecoin(USDT, DAI, FRAX). 
Since, initially the protocol is going to use uniswap pools to ack as an exit(redeem) for zJTT/zSTT tokens the attacker can use flash loans to increase the amount of funds under attack.
And even when the protocol implements OCR contract to redeem tranche tokens, the attack is still possible but just not in one transaction(i.e. no flash loans). 

Example: 
USDC is 50% down(for the sake of our example),
Alice mints deposits 100 USDC and mints 100 zSTT(because amount is used directly instead of oracles) when she should be given only 50 zSTT.
Alice redeems 100 zSTT in USDT(uniswap pool/OCR contract) and gets 100 USDT and profits for 50 dollars which is a loss for innocent users. 

## Impact
Incase one of the stablecoins used depegs(which had happened in the past) it can be used to mint trancheTokens(zJTT/zSTT) and redeem it with another stablecoin for profit.

## Code Snippet
The code that can be exploited under this event are the ```depositJunior``` and ```depositSenior``` functions in both [```ZivoeITO```](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeITO.sol#L248C5-L298C6) and [```ZivoeTranches```](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeTranches.sol#L268C5-L315C6) contracts.

## Tool used

Manual Review

## Recommendation
The simplest mitigation would be to add oracle checks in the ```standardize``` function in ```ZivoeGlobals``` because this is called by all the deposit functions, and revert incase the given stablecoin depegs.
Implement this function:
```solidity
function _checkStable(address asset) internal {
    if(asset == USDC){
        check USDC/USD oracle and revert if not 1:1 with USD
    }
    if(asset == USDT) {
        check USDT/USD oracle and revert if not 1:1 with USD
    }
    if(asset == DAI) {
        check DAI/USD oracle and revert if not 1:1 with USD
    }
    if(asset == FRAX) {
        check FRAX/USD oracle and revert if not 1:1 with USD
    }

    // check chainlink oracles and revert if the given asset is not 1:1 with USD 
}

```

and add this line in the ```standardize``` function:
```diff

+      _checkOracle(asset);
```
