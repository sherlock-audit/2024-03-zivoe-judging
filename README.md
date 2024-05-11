# Issue H-1: Anyone could call `depositReward` with zero reward to extend the period finish time 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/11 

## Found by 
0brxce, 0xAnmol, 0xboriskataa, 0xpiken, 0xvj, 14si2o\_Flint, 9oelm, AMOW, Afriaudit, Bauer, CL001, Dliteofficial, Drynooo, FastTiger, Ironsidesec, Krace, Kunhah, Maniacs, Nihavent, Ruhum, SilverChariot, Timenov, Tychai0s, amar, araj, asui, blockchain555, cergyk, coffiasd, dany.armstrong90, dimulski, forgebyola, heedfxn, jasonxiale, joicygiore, krikolkk, lemonmon, marchev, mt030d, novaman33, pashap9990, rbserver, sakshamguruji, sl1, sunill\_eth, t0x1c
## Summary

Anyone could extend the reward finish time, potentially resulting in users receiving fewer rewards than expected within the same time period.

## Vulnerability Detail

The function `depositReward` can be called by anyone, even with zero rewards, allowing it to be exploited to extend the reward finish time at little cost. 
This could result in loss of rewards; for instance, if there are 10 DAI rewards within a 10-day period, a malicious user could extend the finish time on *day 5*, extending the finish time to the 15th day. Participants would only receive 7.5 DAI by the 10th day.

```solidity
    function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
        IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);

        // Update vesting accounting for reward (if existing rewards being distributed, increase proportionally).
        if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
            rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
        } else {
            uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
            rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
        }

        rewardData[_rewardsToken].lastUpdateTime = block.timestamp;
        rewardData[_rewardsToken].periodFinish = block.timestamp.add(rewardData[_rewardsToken].rewardsDuration);
        emit RewardDeposited(_rewardsToken, reward, _msgSender());
    }
```

### POC
Add the test to `zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol` and run it with `forge test --match-test test_ZivoeRewards_deposit_zero --rpc-url <RPC_URL_MAINNET>`

```diff
diff --git a/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol b/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
index f5353b6..870a531 100644
--- a/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
+++ b/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewards.sol
@@ -685,6 +685,33 @@ contract Test_ZivoeRewards is Utility {

     }

+    function test_ZivoeRewards_deposit_zero() public {
+
+        depositReward_DAI(address(stZVE), 1);
+
+        (
+            uint256 rewardsDuration,
+            uint256 _prePeriodFinish,
+            uint256 _preRewardRate,
+            uint256 lastUpdateTime,
+            uint256 rewardPerTokenStored
+        ) = stZVE.rewardData(DAI);
+        console.log("period finish ", _prePeriodFinish);
+
+        vm.warp(block.timestamp + 1 days);
+
+        depositReward_DAI(address(stZVE), 0);
+
+        (,
+            uint256 _afterPeriodFinish,
+            ,
+            ,
+        ) = stZVE.rewardData(DAI);
+        console.log("period finish ", _afterPeriodFinish);
+        //  extend the Finish 1 day
+        assertEq(_afterPeriodFinish - _prePeriodFinish, 1 days);
+    }
+
     function test_ZivoeRewards_getRewards_works(uint96 random) public {

         uint256 deposit = uint256(random) + 100 ether; // Minimum 100 DAI deposit.
```

## Impact

Anyonce could extend the reward finish time and the users may receive less rewards than expected during the same time period.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243

## Tool used

Foundry

## Recommendation
Only specific users are allowed to call function `depositReward`



## Discussion

**pseudonaut**

Valid, considering adding whitelist

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, allows anyone to extend reward finish time indefinitely and decrease reward rate.



# Issue H-2: ITO can be manipulated 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/12 

## Found by 
0xShax2nk.in, 0xpiken, 0xvj, 14si2o\_Flint, 9oelm, AMOW, Drynooo, Ironsidesec, JigglypuffAndPikachu, KupiaSec, Nihavent, SilverChariot, Tendency, blockchain555, coffiasd, dany.armstrong90, didi, ether\_sky, hulkvision, joicygiore, lemonmon, mt030d, saidam017, sl1, zarkk01
## Summary
The ITO allocates 3 pZVE tokens per senior token minted and 1 pZVE token per junior token minted. When the offering period ends, users can claim the protocol `ZVE token` depending on the share of all pZVE they hold. Only 5% of the total ZVE tokens will be distributed to users, which is equal to `1.25M` tokens.

The ITO can be manipulated because it  uses `totalSupply()` in its calculations.
## Vulnerability Detail
[`ZivoeITO.claimAirdrop()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeITO.sol#L223-L228) calculates the amount of ZVE tokens that should be vested to a certain user. It then creates a vesting schedule and sends all junior and senior tokens to their recipient.

The formula is $pZVEOwned/pZVETotal * totalZVEAirdropped$ (in the code these are called `upper`, `middle` and `lower`).

```solidity
        uint256 upper = seniorCreditsOwned + juniorCreditsOwned;
        uint256 middle = IERC20(IZivoeGlobals_ITO(GBL).ZVE()).totalSupply() / 20;
        uint256 lower = IERC20(IZivoeGlobals_ITO(GBL).zSTT()).totalSupply() * 3 + (
            IERC20(IZivoeGlobals_ITO(GBL).zJTT()).totalSupply()
        );
```

These calculations can be manipulated because they use `totalSupply()`.  The tranche tokens have a public [`burn()`](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeTrancheToken.sol#L72) function.

An attacker can use 2 accounts to enter the ITO. They will deposit large amounts of stablecoins towards the senior tranche. When the airdrop starts, they can claim their senior tokens and start vesting ZVE tokens. The senior tokens can then be burned. Now, when the attacker calls the `claimAirdrop` function with their second account, the denominator of the above equation will be much smaller, allowing them to claim much more ZVE tokens than they are entitled to.

## Impact
There are 2 impacts from exploiting this vulnerability:
 - a malicious entity can claim excessively large part of the airdrop and gain governance power in the protocol
 - since the attacker would have gained unexpectedly large amount of ZVE tokens and the total ZVE to be distributed will be `1.25M`, the users that claim after the attacker may not be able to do so if the amount they are entitled to, added to the stolen ZVE, exceeds `1.25M`.

## Code Snippet
Add this function to [`Test_ZivoeITO.sol`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-testing/src/TESTS_Core/Test_ZivoeITO.sol) and import the `console`.

You can comment the line where Sue burns their tokens and see the differences in the logs.

```solidity
    function test_StealZVE() public {
        // Sam is an honest actor, while Bob is a malicious one
        mint("DAI", address(sam), 3_000_000 ether);
        mint("DAI", address(bob), 2_000_000 ether);
        zvl.try_commence(address(ITO));

        // Bob has another Ethereum account, Sue
        bob.try_transferToken(DAI, address(sue), 1_000_000 ether);
        
        // give approvals
        assert(sam.try_approveToken(DAI, address(ITO), type(uint256).max));
        assert(bob.try_approveToken(DAI, address(ITO), type(uint256).max));
        assert(sue.try_approveToken(DAI, address(ITO), type(uint256).max));

        // Sam deposits 2M DAI to senior tranche and 400k to the junior one
        hevm.prank(address(sam));
        ITO.depositBoth(2_000_000 ether, DAI, 400_000, DAI);
       
        // Bob deposits 2M DAI into the senior tranche using his both accounts
        hevm.prank(address(bob));
        ITO.depositSenior(1_000_000 ether, DAI);

        hevm.prank(address(sue));
        ITO.depositSenior(1_000_000 ether, DAI);

        // Move the timestamp after the end of the ITO
        hevm.warp(block.timestamp + 31 days);
        
        ITO.claimAirdrop(address(sue));
        (, , , uint256 totalVesting, , , ) = vestZVE.viewSchedule(address(sue));

        // Sue burn all senior tokens
        vm.prank(address(sue));
        zSTT.burn(1_000_000 ether);

        console.log('Sue vesting: ', totalVesting / 1e18);

        ITO.claimAirdrop(address(bob));
        (, , , totalVesting, , , ) = vestZVE.viewSchedule(address(bob));

        console.log('Bob vesting: ', totalVesting / 1e18);

        ITO.claimAirdrop(address(sam));
        (, , , totalVesting, , , ) = vestZVE.viewSchedule(address(sam));

        console.log('Sam vesting: ', totalVesting / 1e18);
    }
```


Fair vesting without prior burning
```solidity
  Sue vesting:  312499
  Bob vesting:  312499
  Sam vesting:  625001
```

Vesting after burning
```solidity
  Sue vesting:  312499
  Bob vesting:  416666
  Sam vesting:  833333
```

Bob and Sue will be able to claim `~750 000` ZVE tokens and Sam will not be able to claim any, because the total exceeds `1.25M`.
## Tool used

Manual Review

## Recommendation
Introduce a few new variables in the ITO contract.
```solidity
   bool hasAirdropped;
   uint256 totalZVE;
   uint256 totalzSTT; 
   uint256 totalzJTT;
```

Then check if the call to `claimAidrop` is a first one and if it is, initialize the variables. Use these variables in the vesting calculations.

```solidity
    function claimAirdrop(address depositor) external returns (
        uint256 zSTTClaimed, uint256 zJTTClaimed, uint256 ZVEVested
    ) {
            ...
            if (!hasAirdropped) {
                  totalZVE = IERC20(IZivoeGlobals_ITO(GBL).ZVE()).totalSupply();
                  totalzSTT = IERC20(IZivoeGlobals_ITO(GBL).zSTT()).totalSupply();
                  totalzJTT = IERC20(IZivoeGlobals_ITO(GBL).zJTT()).totalSupply();
                  hasAirdropped = true;
            }
            ...

           uint256 upper = seniorCreditsOwned + juniorCreditsOwned;
           uint256 middle = totalZVE / 20;
           uint256 lower = totalzSTT * 3 + totalzJTT;
      }
```



## Discussion

**pseudonaut**

This is interesting - technically we can support overflow in these situations because there's more ZVE available in other contract. User's would need to burn important tokens, and lock-up capital for extended period of time (more than 1 block) and upside isn't present against what rewards are and capital-loss - net-positive for protocol

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, allows to inflate ITO vesting amounts and cause loss of funds from ITO for the other users



**panprog**

Many dups of this issue talk about naturally increasing `totalSupply` due to new deposits, which decreases airdrop for users who start vesting later. The core reason is the same.

# Issue H-3: User cannot withdraw stakingToken due to incorrect calculation of _totalSupply 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/31 

## Found by 
0xAnmol, 0xvj, 9oelm, AMOW, Afriaudit, Audinarey, Aymen0909, CL001, Dliteofficial, Drynooo, Ironsidesec, JigglypuffAndPikachu, KingNFT, Ruhum, Shield, SilverChariot, Tendency, Varun\_05, amar, araj, asui, beWater0given, denzi\_, dimulski, forgebyola, lemonmon, marchev, mt030d, rbserver, saidam017, sakshamguruji, samuraii77, sl1, supercool, t0x1c, zarkk01
## Summary
User cannot withdraw `stakingToken` due to incorrect calculation of `_totalSupply`

## Vulnerability Detail
Given:

1.userA VestingSchedule` amountToVest`= 20 ether ,`daysToCliff`=30 , `daysToVest`=120

2.userB VestingSchedule `amountToVest`= 30 ether , `daysToCliff`=30, `daysToVest`=120
_totalSupply =20 +30= 50

3.skip 60 days  userB withdraw (`amountWithdrawable =15`)
_totalSupply = 50-15=35

4.`revoke` userB `VestingSchedule` 
_totalSupply = 35-30=5
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L451

5.skip 60 days (120 days later)
userA withdraw `(amountWithdrawable =20`)
_totalSupply = 5-20 lead to revert
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L508


POC:
src/TESTS_Core/Test_ZivoeRewardsVesting.sol
```solidity

    function testpoc() public{
        assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(qcp), 
            30, 
            120,
            20 ether, 
            true
        ));
        assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(pam), 
            30, 
            120,
            30 ether, 
            true
        ));


        hevm.warp(block.timestamp + 60 days);

        vestZVE.amountWithdrawable(address(pam));
        //pam withdraw
         hevm.startPrank(address(pam));
        vestZVE.withdraw();
        hevm.stopPrank();
        vestZVE.totalSupply();

        //revokeVesting
        hevm.startPrank(address(zvl));
        vestZVE.revokeVestingSchedule(address(pam));
        hevm.stopPrank();
        vestZVE.totalSupply();

        //qcp withdraw
        hevm.warp(block.timestamp + 60 days);
        vestZVE.amountWithdrawable(address(qcp));
         hevm.startPrank(address(qcp));
        vestZVE.withdraw();
        hevm.stopPrank(); 
    }
[FAIL. Reason: panic: arithmetic underflow or overflow (0x11)] testpoc() (gas: 795816)
```

## Impact
User cannot withdraw stakingToken

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L451
## Tool used

Manual Review

## Recommendation
```solidity
   function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
        require(
            vestingScheduleSet[account], 
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleSet[account]"
        );
        require(
            vestingScheduleOf[account].revokable, 
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleOf[account].revokable"
        );
        
        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;

        vestingTokenAllocated -= amount;

        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account].totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount - vestingScheduleOf[account].totalWithdrawn);

     -  _totalSupply = _totalSupply.sub(vestingAmount);
     + _totalSupply = _totalSupply.sub(amount);
```



## Discussion

**pseudonaut**

Valid

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high. Incorrect (reduced) totalSupply will make late users being unable to claim their reward



# Issue H-4: Revoking vesting schedule does not subtract user votes correctly 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/118 

## Found by 
0xAnmol, 0xpiken, 0xvj, 9oelm, Afriaudit, Ironsidesec, JigglypuffAndPikachu, KingNFT, KupiaSec, Ruhum, SilverChariot, Tendency, Tychai0s, Varun\_05, amar, denzi\_, dimulski, ether\_sky, lemonmon, marchev, mt030d, rbserver, saidam017, sakshamguruji, samuraii77
## Summary
[ZivoeVestingRewards.revokeVestingSchedule()](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L429-L467) should reduce the voting power of the user with the withdrawable amount plus the revoked amount. However, it reduces it only by the withdrawable amount.

## Vulnerability Detail
When called, `revokeVestingSchedule()` fetches the withdrawable amount by the user at that moment.
```solidity
        uint256 amount = amountWithdrawable(account);
```

The revoke logic is executed and the user's checkpoint value is decreased by `amount`.
```solidity
        _writeCheckpoint(_checkpoints[account], _subtract, amount);
```

The code ignores the amount that's being revoked and the user keeps more voting power than he has to.
Imagine the following:
`totalVested = 1000`
`withdrawable = 0`

If the schedule gets revoked, the user's checkpoint value will not be decreased at all because there is nothing to be withdrawn. The user can later use their voting power to vote on governance proposals.

In fact, `amountWithdrawable(account)` being close to 0 has a very high likelihood because:
   - the user can frontrun the transaction and withdraw the tokens they are entitled to
   -  it's highly likely that a vesting schedule will be removed shortly after creating it.

However, even if `amountWithdrawable()` is not equal to 0, the user would still be left with more voting power.
## Impact
Users keep voting power that must have been taken away.

## Code Snippet
POC to be run in [Test_ZivoeRewardsVesting.sol](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-testing/src/TESTS_Core/Test_ZivoeRewardsVesting.sol)
```solidity
    function test_revoking_leaves_votes() public {
       assert(zvl.try_createVestingSchedule(
            address(vestZVE), 
            address(moe), 
            0, 
            360,
            6000 ether, 
            true
        ));
        // Vesting succeeded
        assertEq(vestZVE.balanceOf(address(moe)), 6000 ether);

        hevm.roll(block.number + 1);
     
        // User votes have increased
        assertEq(vestZVE.getPastVotes(address(moe), block.number - 1), 6000 ether);

        assert(zvl.try_revokeVestingSchedule(address(vestZVE), address(moe)));
        // Revoking succeeded
        assertEq(vestZVE.balanceOf(address(moe)), 0);

        hevm.roll(block.number + 1);
        // User votes have not been decreased at all
        assertEq(vestZVE.getPastVotes(address(moe), block.number - 1), 6000 ether);
    }
```

## Tool used

Foundry

## Recommendation
Subtract the correct amount from the checkpoint's value
```diff
-        _writeCheckpoint(_checkpoints[account], _subtract, amount);
+       _writeCheckpoint(_checkpoints[account], _subtract, vestingAmount - vestingScheduleOf[account].totalWithdrawn + amount)
```




## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, incorrect checkpoints amount subtracted during `revokeVestingSchedule` causes permanent inflated amount of user votes.



# Issue H-5: ````depositReward()```` with zero amount to get reward tokens stuck in ````ZivoeRewards```` contracts 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/129 

## Found by 
Audinarey, Drynooo, KingNFT, KupiaSec, Maniacs, Quanta, SUPERMAN\_I4G, Tricko, beWater0given, dimulski, flacko, joicygiore
## Summary
````ZivoeRewards.depositReward()```` has no access control, attackers can call it with zero amount of ````reward```` to get some reward tokens stuck in contract. The locked amount could be significant especially when reward tokens have small decimals such as ````USDT/USDC/WBTC````.

## Vulnerability Detail
The issue arises due to precision loss on L233 and L237.
```solidity
File: zivoe-core-foundry\src\ZivoeRewards.sol
228:     function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
...
232:         if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
233:             rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
234:         } else {
...
237:             rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
238:         }
...
243:     }

```

The following PoC shows two cases that ````654 USDC```` and ````6.6 WBTC```` get stuck in ````ZivoeRewards(stZVE)```` contract after ````20```` calls on ````depositReward()```` with zero amount of ````reward````.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "../Utility/Utility.sol";
import "forge-std/console2.sol";

contract TestDepositRewardBug is Utility {
    address alice;
    function setUp() public {
        vm.createSelectFork("https://mainnet.gateway.tenderly.co");
        deployCore(false);

        deal(USDC, address(zvl), 1_000e6); // 1,000 USDC
        deal(WBTC, address(zvl), 10e8); // 10 wBTC
        vm.startPrank(address(zvl));
        stZVE.addReward(USDC, 365 days);
        stZVE.addReward(WBTC, 365 days);
        IERC20(USDC).approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(USDC, 1_000e6);
        IERC20(WBTC).approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(WBTC, 10e8);
        vm.stopPrank();


        alice = makeAddr("alice");
        deal(address(ZVE), alice, 1_000e18);
        assertEq(ZVE.balanceOf(alice), 1_000e18);

        vm.startPrank(alice);
        ZVE.approve(address(stZVE), type(uint256).max);
        stZVE.stake(1_000e18);
        vm.stopPrank();

        // alice is the only staker, and is expected to get all rewards
        assertEq(1_000e18, stZVE.totalSupply());
    }

    function testAttackOnUSDCReward() public {
        uint256 timestamp = block.timestamp;
        for (uint256 i; i < 20; ++i) {   
            vm.warp(timestamp += 12);
            // deposit nothing
            stZVE.depositReward(USDC, 0);
        }

        vm.warp(timestamp + 365 days + 1); // make sure all reward distributed
        vm.prank(alice);
        stZVE.getRewards();
        uint256 balance = IERC20(USDC).balanceOf(alice);
        assertApproxEqAbs(346e6, balance, 1e6);
        balance = IERC20(USDC).balanceOf(address(stZVE));
        assertApproxEqAbs(654e6, balance, 1e6);
        console2.log("Expected reward: 1,000 USDC, actual reward: 346 USDC, locked: 654 USDC");
    }

    function testAttackOnWBTCReward() public {
        uint256 timestamp = block.timestamp;
        for (uint256 i; i < 20; ++i) {   
            vm.warp(timestamp += 12);
            // deposit nothing
            stZVE.depositReward(WBTC, 0);
        }

        vm.warp(timestamp + 365 days + 1); // make sure all reward distributed
        vm.prank(alice);
        stZVE.getRewards();
        uint256 balance = IERC20(WBTC).balanceOf(alice);
        assertApproxEqAbs(3.4e8, balance, 0.1e8);
        balance = IERC20(WBTC).balanceOf(address(stZVE));
        assertApproxEqAbs(6.6e8, balance, 0.1e8);
        console2.log("Expected reward: 10 WBTC, actual reward: 3.4 WBTC, locked: 6.6 WBTC");
    }
}
```

The test log:
```solidity
2024-03-zivoe\zivoe-core-testing> forge test --mc TestDepositRewardBug -vv
[⠰] Compiling...
No files changed, compilation skipped

Ran 2 tests for src/TESTS_Core/Bug_DepositReward.t.sol:TestDepositRewardBug
[PASS] testAttackOnUSDCReward() (gas: 822088)
Logs:
  Expected reward: 1,000 USDC, actual reward: 346 USDC, locked: 654 USDC

[PASS] testAttackOnWBTCReward() (gas: 788716)
Logs:
  Expected reward: 10 WBTC, actual reward: 3.4 WBTC, locked: 6.6 WBTC

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 30.37s (5.20s CPU time)

Ran 1 test suite in 30.41s (30.37s CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)

```
## Impact
Fund get stuck in contract

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228C14-L228C27

## Tool used

Manual Review

## Recommendation
Increasing the precision of ````rewardRate```` by ````1e18````.



## Discussion

**pseudonaut**

Valid, considering adding whitelists

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, depositRewards leaves some dust amounts which can accumulate and can be substantial for 6-8 decimals tokens



# Issue H-6: Protocol unable to get extra Rewards in OCY_Convex_C 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/477 

## Found by 
0xpiken, AllTooWell, BoRonGod, Drynooo, Ironsidesec, cergyk, den\_sosnovskyi, ether\_sky
## Summary

Convex would wrap `rewardToken` for pools with IDs 151+, but the counting logic in `OCY_Convex_C.sol` makes it impossible for zivoe to forward yield.

## Vulnerability Detail

In `OCY_Convex_C.sol `, A convex pool with id `270` is used: 

    /// @dev Convex information.
    address public convexDeposit = 0xF403C135812408BFbE8713b5A23a04b3D48AAE31;
    address public convexPoolToken = 0x383E6b4437b59fff47B619CBA855CA29342A8559;
    address public convexRewards = 0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0;

    uint256 public convexPoolID = 270;

In the following logic, `rewardContract` is defaulted to the address of extraRewards. This assumption is fine for pools with PoolId < 150, but would not work for IDs 151+.

    /// @notice Claims rewards and forward them to the OCT_YDL.
    /// @param extra Flag for claiming extra rewards.
    function claimRewards(bool extra) public nonReentrant {
        IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();

        // Native Rewards (CRV, CVX)
        uint256 rewardsCRV = IERC20(CRV).balanceOf(address(this));
        uint256 rewardsCVX = IERC20(CVX).balanceOf(address(this));
        if (rewardsCRV > 0) { IERC20(CRV).safeTransfer(OCT_YDL, rewardsCRV); }
        if (rewardsCVX > 0) { IERC20(CVX).safeTransfer(OCT_YDL, rewardsCVX); }

        // Extra Rewards
        if (extra) {
            uint256 extraRewardsLength = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewardsLength();
            for (uint256 i = 0; i < extraRewardsLength; i++) {
                address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i); 
                //@Audit incorrect here!
                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }
            }
        }
    }


According to [convex doc](https://docs.convexfinance.com/convexfinanceintegration/baserewardpool#:~:text=Important%20for%20pools,wrappers/ConvexStakingWrapper.sol), 

> for pools with IDs 151+:
> VirtualBalanceRewardPool's rewardToken points to a wrapped version of the underlying token.  This Token implementation can be found here: https://github.com/convex-eth/platform/blob/main/contracts/contracts/StashTokenWrapper.sol

Just check `convexRewards` of pool 270: 

https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#readContract#F5

For index 0, it returns a [`VirtualBalanceRewardPool`](https://etherscan.io/address/0x22A0a706Aa423E2257e4217be2268e0374b9229f) with [rewardtoken](https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#readContract#F19) = 0x85D81Ee851D36423A5784CD3Cb6f1a1193Cb5978. This contract is a `StashTokenWrapper`, which is consistent with what the convex documentation says.

And, when `IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();` is triggered, reward tokens will be unwrapped and send to caller, so rewardAmount will always return 0, means such yield cannot be claimed for zivoe.

                address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i); 

                //@Audit incorrect here! `rewardToken` is a `StashTokenWrapper`, not the reward token!

                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }

## Impact

Users will lose extra rewards from convex pools with IDs 151+.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L210-L219
https://docs.convexfinance.com/convexfinanceintegration/baserewardpool

## Tool used

Manual Review

## Recommendation

Change the logic above to:

                uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().token().balanceOf(address(this));
                if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> high, loss of additional reward from convex. Since it accumulates over 30-days period, the loss will be significant. Both `balanceOf` and `safeTransfer` are incorrect - they should be called on `rewardToken().token()`. Different dups of this mention either `balanceOf` or `safeTransfer`, #294 mentions both (but this one is more detailed), but I consider them to be the same root cause, so all are dups.



# Issue M-1: Users are unable to receive any rewards if they are blacklisted by any of the reward tokens 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/13 

The protocol has acknowledged this issue.

## Found by 
AMOW, Aymen0909, DPS, Krace, Timenov, aman, marchev, rbserver
## Summary

Users cannot get any rewards once they are only blacklisted by one `_rewardsTokens`.

## Vulnerability Detail

When claiming rewards, users must claim all rewards for all `_rewardTokens`. If a user is blacklisted by any one of the `_rewardTokens` (such as *USDT*), he cannot receive rewards for other `rewardTokens` because the `safeTransfer` will always revert the `getRewards` function.

```solidity
    /// @notice Claim rewards for all possible _rewardTokens.
    function getRewards() public updateReward(_msgSender()) {
        for (uint256 i = 0; i < rewardTokens.length; i++) { _getRewardAt(i); }
    }
    
    /// @notice Claim rewards for a specific _rewardToken.
    /// @param index The index to claim, corresponds to a given index of rewardToken[].
    function _getRewardAt(uint256 index) internal nonReentrant {
        address _rewardsToken = rewardTokens[index];
        uint256 reward = rewards[_msgSender()][_rewardsToken];
        if (reward > 0) {
            rewards[_msgSender()][_rewardsToken] = 0;
            IERC20(_rewardsToken).safeTransfer(_msgSender(), reward);
            emit RewardDistributed(_msgSender(), _rewardsToken, reward);
        }
    }
```


## Impact

Users cannot get any rewards once they are blacklisted by any one of the `_rewardsTokens`.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L281-L295

## Tool used

Manual Review

## Recommendation

Allow users to get rewards via specified rewardTokens.



## Discussion

**pseudonaut**

Not relevant

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline medium/low, it can be argued that if the user is blacklisted by any 1 reward token, the expected behavior is to block the user from all reward assets as well. But on the other hand, for a true trustless execution, no token should influence the other tokens, so in this sense the user is unable to claim valid tokens he is not blacklisted for, so a loss of funds for the user and valid medium.



# Issue M-2: ZivoeYDL::distributeYield() will revert if protocolRecipients recipients length is smaller than residualRecipients 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/15 

## Found by 
BiasedMerc, dian.ivanov
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



## Discussion

**pseudonaut**

Valid

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, yield distribution always reverts in some cases if protocols recepients < residual recepients



# Issue M-3: OCL_ZVE::pushToLockerMulti() will revert due to incorrect assert() statements when interacting with UniswapV2 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/18 

## Found by 
0rpse, 0x73696d616f, 0xAnmol, 0xboriskataa, 0xpiken, 0xvj, AMOW, Afriaudit, AllTooWell, BiasedMerc, BoRonGod, Drynooo, FastTiger, Ironsidesec, JigglypuffAndPikachu, KupiaSec, Maniacs, Quanta, Ruhum, SilverChariot, Tendency, blackhole, blockchain555, cergyk, dany.armstrong90, den\_sosnovskyi, ether\_sky, flacko, jasonxiale, krikolkk, lemonmon, mt030d, recursiveEth, saidam017, sl1
## Summary

`OCL_ZVE::pushToLockerMulti()` verifies that the allowances for both tokens is 0 after providing liquidity to UniswapV2 or Sushi routers, however there is a high likelihood that one allowance will not be 0, due to setting a 90% minimum liquidity provided value. Therefore, the function will revert most of the time breaking core functionality of the locker, making the contract useless.

## Vulnerability Detail

The DAO can add liquidity to UniswapV2 or Sushi through `OCL_ZVE::pushToLockerMulti()` function, where `addLiquidity` is called on `router`:

[OCL_ZVE.sol#L198C78-L198](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L198)
```solidity
IRouter_OCL_ZVE(router).addLiquidity(
```

[OCL_ZVE.sol#L90](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L90)
```solidity
address public immutable router;            /// @dev Address for the Router (Uniswap v2 or Sushi).
```
The router is intended to be Uniswap v2 or Sushi (Sushi router uses the same code as Uniswap v2 [0xd9e1ce17f2641f24ae83637ab66a2cca9c378b9f](https://etherscan.io/address/0xd9e1ce17f2641f24ae83637ab66a2cca9c378b9f#code)). 

[UniswapV2Router02::addLiquidity](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L61-L76)
```solidity
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
        liquidity = IUniswapV2Pair(pair).mint(to);
    }
```

When calling the function 4 variables relevant to this issue are passed:
`amountADesired` and `amountBDesired` are the ideal amount of tokens we want to deposit, whilst
`amountAMin` and `amountBMin` are the minimum amounts of tokens we want to deposit. 
Meaning the true amount that will deposit be deposited for each token will be inbetween those 2 values, e.g:
`amountAMin <= amountA <= amountADesired`.
Where `amountA` is how much of `tokenA` will be transfered.

The transfered amount are `amountA` and `amountB` which are calculated as follows:
[UniswapV2Router02::_addLiquidity](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L33-L60)
```solidity
    function _addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin
    ) internal virtual returns (uint amountA, uint amountB) {
        // create the pair if it doesn't exist yet
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
    }
```
`UniswapV2Router02::_addLiquidity` receives a quote for how much of each token can be added and validates that the values fall within the `amountAMin` and `amountADesired` range. Unless the exactly correct amounts are passed as `amountADesired` and `amountBDesired` then the amount of one of the two tokens will be less than the desired amount.

Now lets look at how `OCL_ZVE` interacts with the Uniswapv2 router:

[OCL_ZVE::addLiquidity](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L191-L209)
```solidity
        // Router addLiquidity() endpoint.
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
        emit LiquidityTokensMinted(minted, depositedZVE, depositedPairAsset);
        assert(IERC20(pairAsset).allowance(address(this), router) == 0);
        assert(IERC20(ZVE).allowance(address(this), router) == 0);
```
The function first increases the allowances for both tokens to `balPairAsset` and `balZVE` respectively. 

When calling the router, `balPairAsset` and `valZVE` are provided as the desired amount of liquidity to add, however `(balPairAsset * 9) / 10` and `(balZVE * 9) / 10` are also passed as minimums for how much liquidity we want to add.

As the final transfered value will be between:
 `(balPairAsset * 9) / 10 <= x <= balPairAsset`
therefore the allowance after providing liquidity will be:
 `0 <= IERC20(pairAsset).allowance(address(this), router) <= balPairAsset - (balPairAsset * 9) / 10` 
however the function expects the allowance to be 0 for both tokens after providing liquidity.
The same applies to the `ZVE` allowance.

This means that in most cases one of the assert statements will not be met, leading to the add liquidity call to revert. This is unintended behaviour, as the function passed a `90%` minimum amount, however the allowance asserts do not take this into consideration.

## Impact

Calls to `OCL_ZVE::pushToLockerMulti()` will revert a majority of the time, causing core functionality of providing liquidity through the locker to be broken.

## Code Snippet

[OCL_ZVE.sol#L198C78-L198](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L198)
[UniswapV2Router02.sol#L61-L76](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L61-L76)
[UniswapV2Router02.sol#L33-L60](https://github.com/Uniswap/v2-periphery/blob/master/contracts/UniswapV2Router02.sol#L33-L60)
[OCL_ZVE.sol#L191-L209](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L191-L209)

## Tool used

Manual Review

## Recommendation

The project wants to clear allowances after all transfers, therefore set the router allowance to 0 after providing liquidity using the returned value from the router:
```diff
  (uint256 depositedPairAsset, uint256 depositedZVE, uint256 minted) = IRouter_OCL_ZVE(router).addLiquidity(
      pairAsset, 
      ZVE, 
      balPairAsset,
      balZVE, 
      (balPairAsset * 9) / 10,
      (balZVE * 9) / 10, 
      address(this), block.timestamp + 14 days
  );
  emit LiquidityTokensMinted(minted, depositedZVE, depositedPairAsset);
- assert(IERC20(pairAsset).allowance(address(this), router) == 0);
- assert(IERC20(ZVE).allowance(address(this), router) == 0);
+ uint256 pairAssetAllowanceLeft = balPairAsset - depositedPairAsset;
+ if (pairAssetAllowanceLeft > 0) {
+     IERC20(pairAsset).safeDecreaseAllowance(router, pairAssetAllowanceLeft);
+ }
+ uint256 zveAllowanceLeft = balZVE - depositedZVE;
+ if (zveAllowanceLeft > 0) {
+     IERC20(ZVE).safeDecreaseAllowance(router, zveAllowanceLeft);
+ }
```
This will remove the left over allowance after providing liquidity, ensuring the allowance is 0.



## Discussion

**pseudonaut**

Valid

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium. While amounts provided are from the trusted admin, who is supposed to provide exact amounts to add liquidity in correct ratio of token amounts, the fact that live balance of the contract is used makes it possible for the balance to change from the time admin provides the amounts, making the transaction revert. Additionally, current contract balances might be in so high disproportion, that admin will simply not have enough funds to create correct ratio to add liquidity.



# Issue M-4: distributeYield() calls earningsTrancheuse() with outdated emaSTT & emaJTT while calculating senior & junior tranche yield distributions 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/54 

## Found by 
Ironsidesec, Nihavent, t0x1c
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



## Discussion

**pseudonaut**

Not an issue, the yield that accumulated and is for distribution was generated over the last 30 days and thus should match the initial fix-point of the prior 30 day EMA calculation. There's dilution that occurs with this methodology that we don't want to introduce into our overall accounting system

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, distributeYield uses outdated ema values, updating them after all calculations. However, the impact is limited, because the expected EMA averaging period is 2 months and it's unlikely that distributeYield is called so rarely that the impact of outdated ema values causes any real issues.



**panprog**

After considering developer's response and intentions, changing this to invalid.

According to developers response, this is intentional, because using current EMA (which includes current totalSupply at the time of distribution, which might be manipulated) can introduce some attack vectors from the current totalSupply, so current distribution is done over EMA from the previous period.

**panprog**

Sponsor response:
> we did a run-through on accounting scenarios
It is indeed underpaying the depositors
By what we think is fair within the system
And manipulation isn't possible because they'd have to deposit and lock up tokens so there's no flash-loan etc.

Agree with sponsor, changing back to Medium

# Issue M-5: Forwarding yield in `OCL_ZVE` is possible a lot more often than the enforced 30 days 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/55 

## Found by 
0xpiken, Krace, cergyk, lemonmon, samuraii77
## Summary
Calling `OCL_ZVE::forwardYield()` is supposed to only be possible every 30 days however in reality, it is possible to call the function a lot more often.

## Vulnerability Detail
The `require` statement in `OCL_ZVE::forwardYield()` enforces that the function should only be callable when `block.timestamp` is of larger value than the value of `nextYieldDistribution`
```solidity
    function forwardYield() external {
        if (IZivoeGlobals_OCL_ZVE(GBL).isKeeper(_msgSender())) {
            require(
                block.timestamp > nextYieldDistribution - 12 hours, 
                "OCL_ZVE::forwardYield() block.timestamp <= nextYieldDistribution - 12 hours"
            );
        }
        else {
            require(
                block.timestamp > nextYieldDistribution, 
                "OCL_ZVE::forwardYield() block.timestamp <= nextYieldDistribution"
            );
        }

        (uint256 amount, uint256 lp) = fetchBasis();
        if (amount > basis) { _forwardYield(amount, lp); }
        (basis,) = fetchBasis();
        nextYieldDistribution += 30 days;
    }
```
Then, at the end of the function, `nextYieldDistribution` is incremented by 30 days making the function supposedly not callable for another 30 days. However, there is a vulnerability that allows `OCL_ZVE::forwardYield()` to be called a lot more times.

Upon calling the `OCL_ZVE::pushToLockerMulti()`, the `nextYieldDistribution` gets set to `block.timestamp + 30 days` if its value is 0.
```solidity
if (nextYieldDistribution == 0) { nextYieldDistribution = block.timestamp + 30 days; }
```
If this is the way `nextYieldDistribution` gets its first value, then there will not be an issue. However, if `OCL_ZVE::forwardYield()` is called beforehand, then `nextYieldDistribution` will be set to `0 += 30 days` which equals `30 days` making its value a lot less than the value of `block.timestamp` resulting in people being able to call `OCL_ZVE::forwardYield()` at will all the way until `nextYieldDistribution` gets incremented all the way to `block.timestamp`.

Calling `OCL_ZVE::forwardYield()` before any other function is possible and requires just 1 simple circumstance to be a fact.

Imagine the following scenario:
1. `OCL_ZVE::forwardYield()` is called
2. The `if` statement passes as `block.timestamp` is larger than `nextYieldDistribution`, the value of which is the default value of 0
3. `OCL_ZVE::fetchBasis()` gets called
```solidity
function fetchBasis() public view returns (uint256 amount, uint256 lp) {
        address pool = IFactory_OCL_ZVE(factory).getPair(pairAsset, IZivoeGlobals_OCL_ZVE(GBL).ZVE());
        uint256 pairAssetBalance = IERC20(pairAsset).balanceOf(pool);
        uint256 poolTotalSupply = IERC20(pool).totalSupply();
        lp = IERC20(pool).balanceOf(address(this));
        amount = lp * pairAssetBalance / poolTotalSupply;
    } 
```
4. As long as there is a pool setup for `pairAsset` and `ZVE`, the vulnerability will take place
5. Pool gets the value of the pool address
6. The only thing which has to pass here is `poolTotalSupply` not being 0 as division by 0 is not possible
7. That successfully passes as there is already a setup pool for `pairAsset` and `ZVE`
8. Then, we get back to the `OCL_ZVE::forwardYield()` function
9. `if(amount > basis)` does not pass as both values are 0
10. On the last line, `nextYieldDistribution` gets set to `30 days` making the `OCL_ZVE::forwardYield()` function callable again and again

## Impact
`OCL_ZVE::forwardYield()` is callable over and over again even though it is only supposed to be called every 30 days

## Proof Of Concept
Paste the following test into `Test_OCL_ZVE.sol`:
```solidity
function testCanForwardYieldALot() public {
        address UNIV2_ROUTER = OCL_ZVE_UNIV2_DAI.router();
        deal(DAI, address(this), 10000);

        IERC20(DAI).safeApprove(UNIV2_ROUTER, 10000);
        IERC20(ZVE).safeApprove(UNIV2_ROUTER, 1001);

        IUniswapV2Router01(UNIV2_ROUTER).addLiquidity(address(DAI), address(ZVE), 10000, 1001, 0, 0, address(this), block.timestamp);

        uint256 count;
        while (block.timestamp > OCL_ZVE_UNIV2_DAI.nextYieldDistribution()) {
            OCL_ZVE_UNIV2_DAI.forwardYield();
            count++;
        }

        console.log(count);
    }
```

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L186
https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L287-L305

## Tool used

Manual Review

## Recommendation
Do not allow `OCL_ZVE::forwardYield()` to be called whenever the value of `nextYieldDistribution` is equal to 0. 
Also, another option is to set `nextYieldDistribution` to `block.timestamp + 30 days` instead of just incrementing it by `30 days`.



## Discussion

**coreggon11**

If `pushToLockerMulti` was not called, then nothing will be forwarded.

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, if `forwardYield` is called just after contract creation, then `nextYieldDistribution` becomes incorrect and `forwardYield` can be called at any time by anyone. The impact is limited, but the protocol functionality is broken (no time limit for calling `forwardYield`)



**pseudonaut**

Wouldn't `_forwardYield()` revert due to division by 0 if basis is 0 ?

```solidity
function forwardYield() external {
        ...
        (uint256 amount, uint256 lp) = fetchBasis();
        if (amount > basis) { _forwardYield(amount, lp); }
```

Or rather `_forwardYield()` wouldn't even be called `if (amount > basis) { _forwardYield(amount, lp); }` ...

```solidity
function _forwardYield(uint256 amount, uint256 lp) private nonReentrant {
        address ZVE = IZivoeGlobals_OCL_ZVE(GBL).ZVE();
        uint256 lpBurnable = (amount - basis) * lp / amount * (BIPS - compoundingRateBIPS) / BIPS;
```

**panprog**

@pseudonaut 
This is correct, `_forwardYield` will not be called, the point is that the last line of `forwardYield`:
```solidity
nextYieldDistribution += 30 days;
```
will make `nextYieldDistribution = 30 days`, thus all future `forwardYield` will ignore timestamp check because `block.timestamp` will always be greater than `nextYieldDistribution`.

The issue is valid, keeping it medium.

# Issue M-6: When APR late rate is lower than APR, an OCC locker bullet loan borrower can pay way less interests by calling the loan 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/97 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, Ironsidesec, KupiaSec, SilverChariot, saidam017, sl1, thank\_you, y4y
## Summary
A bullet loan borrower can pay less interests by calling [`callLoan`](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCC/OCC_Modular.sol#L492)  at the end of payment period.

## Vulnerability Detail
In `OCC_Modular` contract, the protocol can create loan offers, and users can accept them. The loan has two types, one being bullet, and the other being amortization. In the bullet loan, borrowers only need to pay back interests for each interval, and principle at the last term.

`amountOwed` returns the payment amount needed for each loan id:

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

And we see, there is a `lateFee` for any loans which is overdue. The later the borrower pays back the loan, the more late fees will be accumulated. Plus, the under writer role can always set the loan to default when it's way passed grace period. `callLoan` provides an option for borrowers to payback all he/she owes immediately and settles the loan. In this function, `amountOwed` is called once:

```solidity
    function callLoan(uint256 id) external nonReentrant {
        require(
            _msgSender() == loans[id].borrower || IZivoeGlobals_OCC(GBL).isLocker(_msgSender()), 
            "OCC_Modular::callLoan() _msgSender() != loans[id].borrower && !isLocker(_msgSender())"
        );
        require(loans[id].state == LoanState.Active, "OCC_Modular::callLoan() loans[id].state != LoanState.Active");

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

        IERC20(stablecoin).safeTransferFrom(_msgSender(), owner(), principalOwed);

        loans[id].principalOwed = 0;
        loans[id].paymentDueBy = 0;
        loans[id].paymentsRemaining = 0;
        loans[id].state = LoanState.Repaid;
    }
```

This means, only one interval's late fee is taken into account for this calculation. When the late fee rate is less than APR, and the payment is way overdue, it's possible for such borrower to skip a few interests and late fee payment.

```solidity
    function test_OCC_Late_Payment_Loan_ALL() public {
        uint256 borrowAmount = 20 * 10 ** 18;
        uint256 APR;
        uint256 APRLateFee;
        uint256 term;
        uint256 paymentInterval;
        uint256 gracePeriod;
        int8 paymentSchedule = 0;
        
        APR = 1000;
        APRLateFee = 500;
        term = 10;
        paymentInterval = options[1];
        gracePeriod = uint256(10 days) % 90 days + 7 days;
        mint("DAI", address(tim), 100 * 10 **18);
        hevm.startPrank(address(roy));
        OCC_Modular_DAI.createOffer(
            address(tim), borrowAmount, APR, APRLateFee, term, paymentInterval, gracePeriod, paymentSchedule
        );
        hevm.stopPrank();

        uint256 loanId = OCC_Modular_DAI.loanCounter() - 1;

        hevm.startPrank(address(tim));
        IERC20(DAI).approve(address(OCC_Modular_DAI), type(uint256).max);
        OCC_Modular_DAI.acceptOffer(loanId);
        (, , uint256[10] memory info) = OCC_Modular_DAI.loanInfo(loanId);
        hevm.warp(info[3] + (info[4] * term));
        uint256 balanceBefore = IERC20(DAI).balanceOf(address(tim));
        while (info[4] > 0) {
            OCC_Modular_DAI.makePayment(loanId);
            (, , info) = OCC_Modular_DAI.loanInfo(loanId);
        }
        uint256 balanceAfter = IERC20(DAI).balanceOf(address(tim));
        uint256 diff = balanceBefore - balanceAfter;
        console.log("paid total when payments are late for each interval:", diff);
        console.log("total interests:", diff - borrowAmount);
        hevm.stopPrank();
    }

    function test_OCC_Normal_Payment_Loan() public {
        uint256 borrowAmount = 20 * 10 ** 18;
        uint256 APR;
        uint256 APRLateFee;
        uint256 term;
        uint256 paymentInterval;
        uint256 gracePeriod;
        int8 paymentSchedule = 0;
        
        APR = 1000;
        APRLateFee = 500;
        term = 10;
        paymentInterval = options[1];
        gracePeriod = uint256(10 days) % 90 days + 7 days;
        mint("DAI", address(tim), 100 * 10 ** 18);
        hevm.startPrank(address(roy));
        OCC_Modular_DAI.createOffer(
            address(tim), borrowAmount, APR, APRLateFee, term, paymentInterval, gracePeriod, paymentSchedule
        );
        hevm.stopPrank();

        uint256 loanId = OCC_Modular_DAI.loanCounter() - 1;

        hevm.startPrank(address(tim));
        IERC20(DAI).approve(address(OCC_Modular_DAI), type(uint256).max);
        OCC_Modular_DAI.acceptOffer(loanId);
        (, , uint256[10] memory info) = OCC_Modular_DAI.loanInfo(loanId);
        hevm.warp(info[3]);
        uint256 balanceBefore = IERC20(DAI).balanceOf(address(tim));
        while (info[4] > 0) {
            OCC_Modular_DAI.makePayment(loanId);
            (, , info) = OCC_Modular_DAI.loanInfo(loanId);
            hevm.warp(info[3]);
        }
        uint256 balanceAfter = IERC20(DAI).balanceOf(address(tim));
        uint256 diff = balanceBefore - balanceAfter;
        console.log("paid total when loan is solved normally:", diff);
        console.log("total interests:", diff - borrowAmount);
        hevm.stopPrank();
    }

    function test_OCC_Late_Call_Loan() public {
        uint256 borrowAmount = 20 * 10 ** 18;
        uint256 APR;
        uint256 APRLateFee;
        uint256 term;
        uint256 paymentInterval;
        uint256 gracePeriod;
        int8 paymentSchedule = 0;
        
        APR = 1000;
        APRLateFee = 500;
        term = 10;
        paymentInterval = options[1];
        gracePeriod = uint256(10 days) % 90 days + 7 days;
        mint("DAI", address(tim), 100 * 10 ** 18);
        hevm.startPrank(address(roy));
        OCC_Modular_DAI.createOffer(
            address(tim), borrowAmount, APR, APRLateFee, term, paymentInterval, gracePeriod, paymentSchedule
        );
        hevm.stopPrank();

        uint256 loanId = OCC_Modular_DAI.loanCounter() - 1;

        hevm.startPrank(address(tim));
        IERC20(DAI).approve(address(OCC_Modular_DAI), type(uint256).max);
        OCC_Modular_DAI.acceptOffer(loanId);
        (, , uint256[10] memory info) = OCC_Modular_DAI.loanInfo(loanId);
        hevm.warp(info[3] + (info[4] * term));
        uint256 balanceBefore = IERC20(DAI).balanceOf(address(tim));
        OCC_Modular_DAI.callLoan(loanId);
        uint256 balanceAfter = IERC20(DAI).balanceOf(address(tim));
        uint256 diff = balanceBefore - balanceAfter;
        console.log("paid total when use `callLoan` at the end:", diff);
        console.log("total interests:", diff - borrowAmount);
        hevm.stopPrank();
    }
```

In the above test cases, all three of them will have the same borrower, and borrow the same loan, with same details and everything. One of them simulating when a borrower pays all charges normally till the end of term, another one waits till the very end to pay back the loan with late fees, and the last one also wait till the end, except calls `callLoan` to settle the loan instead of normally paying back each interval's amount.

After running the test cases, the following will be logged:

```plaintext
[PASS] test_OCC_Late_Call_Loan() (gas: 373730) 
Logs:
  paid total when use `callLoan` at the end: 20076715499746321663
  total interests: 76715499746321663       

[PASS] test_OCC_Late_Payment_Loan_ALL() (gas: 565561)  
Logs:
  paid total when payments are late for each interval: 20767126458650431246
  total interests: 767126458650431246  

[PASS] test_OCC_Normal_Payment_Loan() (gas: 569173)
Logs:
paid total when loan is solved normally: 20767123287671232870
total interests: 767123287671232870
```

As we can see, while `callLoan` also needs to pay the late fee penalty, it still charges way less than normally paying back the loan. This makes a borrower being able to skip a few interests fee, with the cost of little late fees.

## Impact
The PoC provided above is certainly an exaggerated edge case, but it's also possible when late fees are aribitrary, as long as the loan is not set to default by under writers, the borrower can skip paying quite some interest fees by exploiting this at the cost of a few late fees. This is more likely to happen when intervals are set to 7 days, as the minimum grace period is 7 days.


## Code Snippet
```solidity
    function callLoan(uint256 id) external nonReentrant {
        require(
            _msgSender() == loans[id].borrower || IZivoeGlobals_OCC(GBL).isLocker(_msgSender()), 
            "OCC_Modular::callLoan() _msgSender() != loans[id].borrower && !isLocker(_msgSender())"
        );
        require(loans[id].state == LoanState.Active, "OCC_Modular::callLoan() loans[id].state != LoanState.Active");

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

        IERC20(stablecoin).safeTransferFrom(_msgSender(), owner(), principalOwed);

        loans[id].principalOwed = 0;
        loans[id].paymentDueBy = 0;
        loans[id].paymentsRemaining = 0;
        loans[id].state = LoanState.Repaid;
    }
```

## Tool used

Manual Review, foundry

## Recommendation
Prohibits bullet loan borrowers from calling `callLoan` when the loan is late and still has more than 1 intervals to finish.



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, if there are more than 1 interest payments missed by borrower, then `callLoan` takes only 1 period payment, allowing borrower to skip paying the other periods interest and lateFee payments.



**pseudonaut**

This is not a valid issue - calling a loan is much different than making a payment as it requires the full amount of principal vs. (in the case of bullet loans) a single payment with interest only


**panprog**

Yes, it's much different from making a payment, but it still allows to bypass paying additional interests/late fees, for example, when only a few payment periods are remaining and it makes sense to simply `callLoan` instead of doing last 2-3 interest and latefee payments at the expense of protocol / depositors.

Keeping this as medium.

**panprog**

Sponsor response:
> intended functionality, user has option to callLoan at any point in time, and if you're saying they have late-fee's than theoretically the loan could be defaulted and likely would at that point preventing callLoan

**panprog**

This is from [docs](https://docs.zivoe.com/user-docs/borrowers/how-do-defaults-work):
> In some cases, the grace period may be longer than the payment interval, and the borrower may miss several loan payments before a loan enters default. In such an event, the borrower must resolve each missed payment before late fees stop accruing. 

So it's possible that grace period is longer than payment interval. Example: payment interval = 7 days, grace period = 28 days.
Borrower can simply stop paying 28 days before the loan is due. At the end of the loan he will have 4 missed payments, at which point he simply `callLoan` and pay only 1 missed payment + late fees from it, skipping paying the other 3 missed payments and late fees from them. Depending on late fees this can be cheaper for borrower than paying all 4 payments on time.
Scenario A (paying on time): 28 days * APR payments
Scenario B (calling loan in the end): 7 days * APR +  28 days * lateFee APR

If lateFee is less than APR, then borrower is better off skipping the last 4 payments and doing `callLoan` in the end.

Keeping this medium.

# Issue M-7: Rewards are calculated as distributed even if there are no stakers, locking the rewards forever 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/113 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, BoRonGod, Ironsidesec, SUPERMAN\_I4G, Tendency, dimulski, marchev, rbserver
## Summary
The ``ZivoeRewards.sol`` and the ``ZivoeRewardsVesting.sol`` contracts are a fork of the Synthetix rewards distribution contract, with slight modifications. The contract logic keeps track of the total time passed, as well as the cumulative rate of rewards that have been generated for each token used for rewards, so that for every user, it just has to keep track of the last timestamp and last rate at the time of the last reward withdrawal, stake, or unstake in order to calculate the rewards owed since the user began staking. The code special-cases the scenario where there are no users, by not updating the cumulative rate when the _totalSupply is zero, but it does not include such a condition for the tracking of the timestamp. Because of this, even when there are no users staking, the accounting logic still thinks funds were being dispersed during that timeframe (because the starting timestamp is updated), which means the funds effectively are distributed to nobody, rather than being saved for when there is someone to receive them. And will be locked in the contract forever. One of the modifications is this line in the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function. 
```solidity
      IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);
```
Contrary to the Synthetix implementation, Zivo requires each time reward is deposited to the contact, the reward amount to be transferred in the same transaction. However if a reward is deposited but there are no stakers, the reward that should have been distributed for the period until the first user stakes, will be locked in the contract forever, and won't be able to be added to the rewards for the next reward period because of the above code snippet. 

## Vulnerability Detail
[Gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb)
After following the steps in the above linked [gist](https://gist.github.com/AtanasDimulski/e2edba2c03e4dd1325b9e73c8fd58ddb) add the following test to the ``AuditorTests.t.sol`` contract:

```solidity
    function test_LockedRewards() public {
        vm.startPrank(ZVL);
        zivoeToken.transfer(alice, 5_000_000e18);
        uint256 duration = 30 days;
        stZVE.addReward(address(mockUSDC), duration);
        vm.stopPrank();

        vm.startPrank(simulateYLD);
        mockUSDC.mint(simulateYLD, 50_000e6); // this represent 50_000 USDC tokens to be distributed in the next 30 days
        mockUSDC.approve(address(stZVE), type(uint256).max);
        stZVE.depositReward(address(mockUSDC), 50_000e6);
        skip(172_800); /// @notice two days pass, before anybody stakes in the contract
        vm.stopPrank();

        vm.startPrank(alice);
        console2.log("USDC balance of alice before staking: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract: ", mockUSDC.balanceOf(address(stZVE)));
        zivoeToken.approve(address(stZVE), type(uint256).max);
        stZVE.stake(5_000_000e18);
        skip(duration);
        stZVE.fullWithdraw();
        console2.log("USDC balance of alice after staking for 30 days: ", mockUSDC.balanceOf(alice));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period: ", mockUSDC.balanceOf(address(stZVE)));
        console2.log("USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized: ", mockUSDC.balanceOf(address(stZVE)) / 1e6);
        console2.log("zivoToken balance of alice after unstaking: ", zivoeToken.balanceOf(alice));
        vm.stopPrank();
    }
```

```solidity
Logs:
  USDC balance of alice before staking:  0
  USDC balance of stZVE contract:  50000000000
  USDC balance of alice after staking for 30 days:  46665000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period:  3335000000
  USDC balance of stZVE contract, when users have withdrawn everything for the first period normalized:  3335
  zivoToken balance of alice after unstaking:  5000000000000000000000000
```
As can be seen from the above logs **3335 USDC tokens** will be locked forever. 

To run the test use: ``forge test -vvv --mt test_LockedRewards``
## Impact
If the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function is called prior to there being any users staking, the funds that should have gone to the first stakers will instead accrue to nobody, and be locked in the contract forever. 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243

## Tool used
Manual Review & Foundry

## Recommendation
Remove the ``safeTransfer()`` from the [depositReward()](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeRewards.sol#L228-L243) function, and instead check if there are enough reward tokens already in the contract. Something similar to the [Synthetix implementation](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol#L113-L132).



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline medium/low. If there are no stakers and depositReward is called, the deposit amount might be lost since it isn't distributed to anybody.



**panprog**

Keeping this medium as the loss of funds can happen, although the probability of this is very low, but still possible.

# Issue M-8: EMA Data Point From Unlock Is Discarded 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/201 

## Found by 
JigglypuffAndPikachu, Tendency, lemonmon, pseudoArtist
## Summary

The `ema` calculation fails to incorporate the `emaJTT` and `emaSTT` amounts which are initally set in `unlock`.

## Vulnerability Details

Let's carefully consider the EMA calculation. Firstly during the unlock, the `emaSTT` and `emaJTT `are set to the current values. This acts as our first data point. Since this is the only data point, this doesn't have any averaging which is why `MATH.ema()` is not called.

Now, after unlocking consider when `distributeYield` is called for the first time after unlocking. `emaSTT` and `emaJTT` should incorporate the first data point (which was recorded during unlock) with the new `totalSupply` of `STT` and `JTT`.

Now let's show why the contract will actually ignore the `emaSTT`/JTT set during unlock during the first yield distribution:

The `distributionCounter` is `0` when `distributeYield` is called, but due to the line `distributionCounter += 1;`, `1` is passed as the 3rd parameter to the `ema` formula in `emaSTT = MATH.ema(emaSTT, aSTT, retrospectiveDistributions.min(distributionCounter));`

When `N == 1` in the `ema` formula, the function will return only `eV`, which is only the latest data point. It completely ignores `bV` which is the data point during the unlock:

```solidity
function ema(uint256 bV, uint256 cV, uint256 N) external pure returns (uint256 eV) {

    assert(N != 0);

    uint256 M = (WAD * 2).floorDiv(N + 1); //When N = 1, M = WAD

    eV = ((M * cV) + (WAD - M) * bV).floorDiv(WAD); //Substituting M = WAD, eV = cV

}
```

## Impact

Incorrect EMA calculation which leads to incorrect yield distribution

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L213-L310

## Tool used

Manual Review

## Recommendation

The `distributionCounter` should actually be `2` rather than 1 during the first pass into the `MATH.ema()` formula. However then the varibale would not really correspond to the number of times `distribution` was called, so it might need to be renamed.



## Discussion

**pseudonaut**

Valid, distributionCounter must be set to 1 initially

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline low/medium, while true, the impact is questionable as there can be situations when current method is better than fixed one, so this is basically a choice of N=1 or N=2 - both are valid, just whatever developer prefers to better fit the protocol needs.



**panprog**

Keeping this medium as sponsor confirmed it's valid (thus considering that N=2 is the intended value, but it's N=1 right now). The impact then is unintended values used in further calculations.

# Issue M-9: Time calculation issues with exponential decay 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/282 

The protocol has acknowledged this issue.

## Found by 
AMOW, SilverChariot
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



## Discussion

**pseudonaut**

forwardEmissions can be called prior to reset, not an issue

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, OCE_ZVE forwardEmission will start emission as if some large time has already passed due to absence of "start emission" function (to reset lastDistribution timestamp). While it's possible to reset it by calling `forwardEmission`, it requires a minimum of 100e18 tokens, which will have to be distributed to reset the timestamp, which is still a loss of funds. As this is mostly a 1-time action, the impact is limited.



**panprog**

Keeping this medium, because a distribution of 100e18 tokens is still required to reset the `lastDistribution` timestamp.

# Issue M-10: OCL_ZVE.sol::forwardYield relies on manipulable Uniswap V2 pool reserves leading to theft of funds 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/296 

The protocol has acknowledged this issue.

## Found by 
0xpiken, 0xvj, AllTooWell, Audinarey, DPS, Drynooo, JigglypuffAndPikachu, Maniacs, SilverChariot, cergyk, lemonmon, saidam017, t0x1c
## Summary

`OCL_ZVE.sol::forwardYield` is used to forward yield in excess of the basis but it relies on Uniswap V2 pools that can be manipulable by an attacker to set `basis` to a very small amount and steal funds.

## Vulnerability Detail

`fetchBasis` is a function which returns the amount of pairAsset (USDC) which is claimable by burning the balance of LP tokens of OCL_ZVE

`fetchBasis` is called first in `forwardYield` in order to determine the yield to distribute:

[OCL_ZVE.sol#L301-L303](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L301-L303)
```solidity
>>    (uint256 amount, uint256 lp) = fetchBasis();
        if (amount > basis) { _forwardYield(amount, lp); }
        (basis,) = fetchBasis();
```

If the returned `amount` is `<= basis` (negative yield) no yield is forwarded and basis is updated to the low `amount` value 

By manipulating the Uniswap V2 pool with a flashloan, an attacker can get the `uint256 amount` value to be returned by `OCL_ZVE.sol::fetchBasis` to be very small and set `basis` to this very small value:

[OCL_ZVE.sol#L301-L303](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L301-L303)
```solidity
        if (amount > basis) { _forwardYield(amount, lp); }
>>      (basis,) = fetchBasis();
```

The next call (30 days later) to `OCL_ZVE.sol::forwardYield` will forward much more yield (in excess of `basis`) than it should, leading to a loss of funds for the protocol.

### Scenario

1. Attacker buys a very large amount of USDC in the Uniswap V2 pool `ZVE/pairAsset` (can use a flash-loan if needed)
2. Attacker calls `OCL_ZVE.sol::forwardYield`
  - `OCL_ZVE.sol::fetchBasis` returns an incorrect and very small value for `amount`:
  -  No yield is forwarded since `amount < basis`
  - `OCL_ZVE.sol::forwardYield` sets `basis` to `amount`
3. Attacker backruns the calls to `OCL_ZVE.sol::forwardYield` with the sell of his large buy in the Uniswap V2 pool
4. 30 days later, Attacker calls `OCL_ZVE.sol::forwardYield` and steal funds in excess of `basis`

## Impact

Loss of funds for the protocol.

## Code Snippet

- https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L302-L303
- https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L311-L330

## Tool used

Manual Review

## Recommendation
Consider reverting during a call to `forwardYield` if `amount <= basis`



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, allows to distribute almost entire principal as if it's yield, leaving the locker with almost no funds. Since `forwardYield` will usually be called by keepers, who will use flashbots thus making frontrunning out of scope, this is medium. User can still call `forwardYield` but only if keeper didn't call it in the 12 hours prior to end of period, so this requires additional conditions.



**panprog**

Keeping this medium, because `forwardYield` can still be called by user in some circumstances, thus making the attack possible. Even though the distribution is within the protocol, it still distributes inflated amounts to tranches and/or residual recepients, causing accounting issues for the protocol.

**panprog**

Sponsor response:
> it's protected by keeper calls for 12 hours prior and if not allows public accessibility, obviously mev-mitiation is implemented with front-running in mind and then allows public accessibility, intended functionality

**panprog**

It is still possible that user calls this with front-running, even though the probability is very low (keeper has to be down for 12 hours, there might be high network congestion for 12+ hours with keeper being unable to submit transaction due to high gas price etc). This is Medium by Sherlock rules (possible, but very low probability of happening).

# Issue M-11: ZivoeYDL::distributeYield yield distribution is flash-loan manipulatable 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/298 

The protocol has acknowledged this issue.

## Found by 
cergyk
## Summary

`ZivoeYDL::distributeYield` is used to "distributes available yield within this contract to appropriate entities" but it relies on the tokens `totalSupply()` which can be manipulable through a flashloan.

## Vulnerability Detail

`ZivoeYDL::distributeYield` relies on `stZVE().totalSupply()` to distribute protocol earnings and residual earnings:

[ZivoeYDL.sol#L241-L310](https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeYDL.sol#L241-L310)
```solidity
        // Distribute protocol earnings.
...
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                uint256 splitBIPS = (
>>                  IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                ) / (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                );
...
        // Distribute residual earnings.
...
                else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                    uint256 splitBIPS = (
>>                      IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                    ) / (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                        IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                    );
...
```

This can be abused by an attacker by buying then staking a very large amount of `ZVE` right before calling `ZivoeYDL::distributeYield` (a flashloan can be used) in order to game the system and collect a lot more distributed yields than he should be entitled to.

### Scenario

1. Attacker buys and stakes a very large amount of `ZVE` through a flashloan
2. Attacker calls `ZivoeYDL::distributeYield`
3. Attacker collects a very large amount of distributed yields
4. Attacker withdraws and sells back his `ZVE` tokens effectively stealing undeserved yields

## Impact

A user can systematically claim a big chunk of the rewards reserved to `stZVE` and `vestZVE` in `ZivoeYDL` by using a flash loan

## Code Snippet

- https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeYDL.sol#L241-L310

## Tool used

Manual Review

## Recommendation

Use a minimal time locking for stZVE such as it would not be possible to stake and unstake all in one block, or use past (1 block in the past) total supply to claim rewards



## Discussion

**pseudonaut**

Not valid

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium. The scenario presented isn't actually possible, because staking reward is deposited to be distributed over the long period of time, so the attacker won't earn anything from staking for 0 blocks. However, it still allows to manipulate the ratio of stZVE/vestZVE reward allocation, so it's possible to inflate reward to stZVE, then withdraw, but stZVE will receive inflated reward (where attacker can have a significant amount of token staked to benefit from increased yield). This is at the expense of vestZVE which will receive deflated reward.



**panprog**

Keeping this medium.

While the scenario is not possible as described, it still describes a valid issue, attack vector and core reason (manipulation to inflate stZVE's share of distribution).

**panprog**

Sponsor response:
> there's no economically feasible way to make this occur, given restricted circulating supply (<10% circulating for a number of years, with 30% in DAO and 60% in vestZVE, slippage/fees on DEXs and likely supply lowered due to CEX trading) and it implies that user maintains amount being staked after the flash-loan (separate from whatever was used to manipulate it) and doesn't even account for other users coming in over next 30 days and diluting any additional yield earned

**panprog**

You don't need a lot of ZVE to impact the reward distribution. For example, the ratio is 1% in stZVE and 99% in vestZVE. If you buy 1% of ZVE, stake it, call distributeYield, then unstake and sell back ZVE, the ratio is now 2% in stZVE, 98% in vestZVE, so a double rate for stZVE for the next 30 days. The loss for vestZVE is not large, but still loss nonetheless.

Keeping this medium.

# Issue M-12: Unbalanced and imprecise stZVE and vestZVE token distribution occurs inside the ZivoeRewards contract due to the precision loss 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/416 

## Found by 
Matin, Negin
## Summary
Less accurate stZVE and vestZVE tokens are calculated and distributed to the ZivoeRewards contract due to the priority of division over multiplication.

## Vulnerability Detail
Solidity rounds down the result of an integer division, and because of that, it is always recommended to multiply before 
dividing to avoid that precision loss. In the case of a prior division over multiplication, the final result may face serious precision loss
as the first answer would face truncated precision and then multiplied to another integer.

The problem arises in the ZivoeYDL's `distributeYield()` part. This function is responsible for distributing available yield. 
If we look deeply at this function, we can see how the allocated stZVE and vestZVE are calculated for the situation where the recipient is equal to `IZivoeGlobals_YDL(GBL).stZVE()`:

```Solidity
    function distributeYield() external nonReentrant {
        ...
            // Calculation of the protocol earnings
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                uint256 splitBIPS = (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                ) / (
                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                );
                uint stZVEAllocation = _protocol[i] * splitBIPS / BIPS;
                uint vestZVEAllocation = _protocol[i] * (BIPS - splitBIPS) / BIPS;
                IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).stZVE(), stZVEAllocation);
                IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).vestZVE(),vestZVEAllocation);
                IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).stZVE()).depositReward(distributedAsset, stZVEAllocation);
                IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).vestZVE()).depositReward(distributedAsset, vestZVEAllocation);

        ...

            // Calculation of the residual earnings
                else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
                    uint256 splitBIPS = (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
                    ) / (
                        IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
                        IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
                    );
                    uint stZVEAllocation = _residual[i] * splitBIPS / BIPS;
                    uint vestZVEAllocation = _residual[i] * (BIPS - splitBIPS) / BIPS;
                    IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).stZVE(), stZVEAllocation);
                    IERC20(distributedAsset).safeIncreaseAllowance(IZivoeGlobals_YDL(GBL).vestZVE(), vestZVEAllocation);
                    IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).stZVE()).depositReward(distributedAsset, stZVEAllocation);
                    IZivoeRewards_YDL(IZivoeGlobals_YDL(GBL).vestZVE()).depositReward(distributedAsset, vestZVEAllocation);

        ...
    }
```

As it is illustrated, for the case where the recipient is `IZivoeGlobals_YDL(GBL).stZVE()`, the variables `stZVEAllocation`, and `vestZVEAllocation` are distributed to the mentioned recipient (for both
situations of protocol earnings or residual earnings).

These variables are calculated using the preceding formula: (just considering the protocol earning part that is similar to the residual earning part)

allocated stZVE formula:

$$ stZVE = \frac{protocol \ earnings \times splitBIPS}{BIPS} $$


allocated vestZVE formula:

$$ vestZVE = \frac{protocol \ earnings \times (BIPS - splitBIPS)}{BIPS} $$

At its heart, the `splitBIPS` is calculated as the division of stZVE total supply over the sum of stZVE and vestZVE total supplies:

$$ splitBIPS = \frac{stZVE \ totalSupply}{stZVE \ totalSupply + vestZVE \ totalSupply} $$

we can see in the actual implementation, there is a [hidden division before multiplication](https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L250-L257) in the calculation of the `stZVEAllocation`, and `vestZVEAllocation` that rounds down the whole expression. This is bad as the precision loss can be significant, which leads to the pool calculating less accurate `stZVE`, and `vestZVE` than actual.

At the Proof of Concept part, we can check this behavior precisely.

You can run this code to see the difference between the results:

```Solidity
    function test_precissionLoss() public {

        uint stZVEtotalSupply = IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply();
        uint vestZVEtotalSupply = IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply();

        uint256 splitBIPS = (stZVEtotalSupply * BIPS) / 
        (
            stZVEtotalSupply + 
            vestZVEtotalSupply
        );
        uint stZVEAllocation = _protocol * splitBIPS / BIPS;
        uint vestZVEAllocation = _protocol * (BIPS - splitBIPS) / BIPS;

        uint stZVEAllocation_accurate = (_protocol * stZVEtotalSupply * BIPS) / (BIPS * (stZVEtotalSupply + vestZVEtotalSupply));
        uint vestZVEAllocation_accurate = (_protocol * vestZVEtotalSupply * BIPS) / (BIPS * (stZVEtotalSupply + vestZVEtotalSupply));
        
        console.log("Current Implementation of allocated stZVE  ", stZVEAllocation);
        console.log("Current Implementation of allocated vestZVE", vestZVEAllocation);

        console.log("Accurate Implementation of allocated stZVE  ", stZVEAllocation_accurate);
        console.log("Accurate Implementation of allocated vestZVE", vestZVEAllocation_accurate);
    }
```

The result would be: 

(for these sample but real variables: 

`stZVEtotalSupply = 6.62423125 ether`, 

`vestZVEAllocation = 4.5621235125 ether`,

`_protocol = 1563214500000`
)

```Solidity

     Current Implementation of allocated stZVE  : 925579305450
     Current Implementation of allocated vestZVE: 637635194550

     Accurate Implementation of allocated stZVE  : 925689785565
     Accurate Implementation of allocated vestZVE: 637524714434 
```
Thus, we can see that the actual implementation produces **less** allocated stZVE amount and **more** allocated vestZVE amount than the precise method.
The error becomes significant especially when the protocol earning becomes bigger.

## Impact
Precision loss inside the `distributeYield()` function leads to unbalanced stZVE and vestZVE token distributions and fund loss 

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L250-L257
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeYDL.sol#L288-L298

## Tool used

Manual Review

## Recommendation
Consider modifying the allocated tokens calculation to prevent such precision loss and prioritize multiplication over division:

```diff
            else if (_recipient == IZivoeGlobals_YDL(GBL).stZVE()) {
-                uint256 splitBIPS = (
-                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS
-                ) / (
-                    IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
-                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()
-                );
-                uint stZVEAllocation = _protocol[i] * splitBIPS / BIPS;
-                uint vestZVEAllocation = _protocol[i] * (BIPS - splitBIPS) / BIPS;
+                uint stZVEAllocation = _protocol[i] * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() * BIPS) /
+                    (BIPS * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
+                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()));
+                uint vestZVEAllocation = _protocol[i] * (IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply() * BIPS) /
+                    (BIPS * (IERC20(IZivoeGlobals_YDL(GBL).stZVE()).totalSupply() + 
+                    IERC20(IZivoeGlobals_YDL(GBL).vestZVE()).totalSupply()));
```



## Discussion

**pseudonaut**

Valid, could optimize this further

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline low/medium, the loss is at most 1 BIPS which is 1/10000 of the amount, which is not significant, but not tiny either, so borderline issue



# Issue M-13: It is not possible to retrieve some reward tokens ever from `OCY_Convex_C` and `OCY_Convex_A` 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/506 

## Found by 
Ironsidesec, Tricko
## Summary
The issue cannot be low, but medium because the roles of external integration are restricted. So, the `rewardManager` role of `convex` contracts are not trusted. They have the power to add or remove the reward tokens from the convex reward contarcts. So if they remove any reward tokens before the `OCY_Convex_C` claims its rewards, then these rewards are wasted permanently on `OCY_Convex_C` and `OCY_Convex_A`.

Look at the simple recommendation that can solve this issue.
It is not possible to retrieve some tokens ever. It is due to the lack of feature to move tokens from locker to YDL / DAO.

## Vulnerability Detail

From https://github.com/sherlock-audit/2024-03-zivoe?tab=readme-ov-file#q-are-the-admins-of-the-protocols-your-contracts-integrate-with-if-any-trusted-or-restricted

```md
Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?

RESTRICTED
```

https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L200-L220

```solidity

File: 2024-03-zivoe\zivoe-core-foundry\src\lockers\OCY\OCY_Convex_C.sol

200:     function claimRewards(bool extra) public nonReentrant {
201:         IBaseRewardPool_OCY_Convex_C(convexRewards).getReward();
202:
203:         // Native Rewards (CRV, CVX)
204:         uint256 rewardsCRV = IERC20(CRV).balanceOf(address(this));
205:         uint256 rewardsCVX = IERC20(CVX).balanceOf(address(this));
206:         if (rewardsCRV > 0) { IERC20(CRV).safeTransfer(OCT_YDL, rewardsCRV); }
207:         if (rewardsCVX > 0) { IERC20(CVX).safeTransfer(OCT_YDL, rewardsCVX); }
208:
209:         // Extra Rewards
210:         if (extra) {
211:            
212:             uint256 extraRewardsLength = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewardsLength();
213:             for (uint256 i = 0; i < extraRewardsLength; i++) {
214:  >>>            address rewardContract = IBaseRewardPool_OCY_Convex_C(convexRewards).extraRewards(i);
215:  >>>            uint256 rewardAmount = IBaseRewardPool_OCY_Convex_C(rewardContract).rewardToken().balanceOf(
216:                     address(this)
217:                 );
218:                 if (rewardAmount > 0) { IERC20(rewardContract).safeTransfer(OCT_YDL, rewardAmount); }
219:                
220:             }
221:         }
222:     }

```


**Issue path:**

1. The `rewardManager` which is a restricted external  role of `convexRewards`, will add/remove a new reward token and it will stream new reward tokens for may be 7 days or even 2 - 4 weeks. And after a while, that reward token can be removed by that manager.

scroll a bit after going to https://etherscan.io/address/0xc583e81bB36A1F620A804D8AF642B63b0ceEb5c0#code#L835

![image](https://github.com/sherlock-audit/2024-03-zivoe-ironsidesec/assets/162350329/ba1353dd-a7ac-4bd7-a084-8e4f5223d645)



2. But `claimRewards`  was not called in any of these times so reward is not claimed, or maybe the reward manager has called `clearExtraRewards` before this claimreward call, which will remove the reward token. So there's no way `OCY_Convex_C` can claim reward except someone will call `getReward` externally with `OCY_Convex_C` as input address, and reward pending tokens will be transfered.

3. Still, the tokens cannot be transferred to YDL because the token addresses are called from line 214 above, which can be manipulated by `rewardManager`. so effectively wasted rewards.


## Impact
Loss of reward tokens in 2 convex integration contracts A and C. So medium.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_C.sol#L200-L220

https://github.com/sherlock-audit/2024-03-zivoe/blob/01e00e6f27b58392a6fa0b82c84a46a783a0df3c/zivoe-core-foundry/src/lockers/OCY/OCY_Convex_A.sol#L246

## Tool used

Manual Review

## Recommendation

Add the below function to both `OCY_Convex_C` and `OCY_Convex_A` contracts. So if there's any extra reward token in these contracts, they can be transferred out.

```solidity
    function canPullMulti() public override pure returns (bool) { return true; }

```

   



## Discussion

**pseudonaut**

Not of concern

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline medium/low. If external restricted admin (convex pool reward manager) clears reward list, then OCY_Convex_C locker will be unable to get these rewards. It's still possible to call getReward on these removed rewards, and the reward token will be transferred to locker, but there are no functions for admin to retrieve these tokens from the locker. This is extremely unlikely to happen and the loss is one-off and very limited, but still the loss of funds nonetheless.



**panprog**

Keeping this medium as admins of protocols integrated with (Convex) are restricted, thus the scenario described is valid even if not very likely. The loss of funds will be real.

**panprog**

Sponsor response:
> this is really niche, it's one thing if external admin's are doing something exploitative (like adding users to blacklists), but if they're simply managing the rewards in their own protocol and it's intended functionality in their protocol that they can add/remove rewards which leads to us not receiving them, than that's the protocol we're integrating with. it's not malicious or an issue if that's how their protocol works

**panprog**

The issue is that convex reward admin can be well-intended by removing/changing rewards. This doesn't stop anyone from still claiming this reward (but manually, not via the same functions as normally). Anyone can claim it, and it will be transferred to locker address. However, OCY_CONVEX_A/C lockers do not have any functions to send these claimed tokens anywhere, so they will be stuck in the locker. These lockers override `pullFromLocker` to only pull from convex, but at this time convex won't have these rewards, they'll be directly in locker.
Keeping this as medium.

# Issue M-14: Title: Inadequate Allowance Handling in convertAndForward Function of `OCT_DAO` & `OCT_YDL`. 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/609 

## Found by 
0xBhumii, recursiveEth
## Summary
The `OCT_DAO:convertAndForward `and `OCT_YDL:convertAndForward `function suffers from inadequate handling of token allowances for the 1inch router. Although allowances are set correctly before converting assets, they are not reset afterward. This can lead to failed transactions if the 1inch router does not utilize the entire allowance due to slippage or other factors.

## Vulnerability Detail

the issue arises from the lack of allowance reset after interacting with the 1inch router. If the router does not utilize the entire allowance specified due to slippage or other reasons, the allowance will remain unchanged, potentially causing the assertion to fail and the transaction to revert.

## Impact

The impact of this vulnerability is that transactions may fail due to incorrect allowance management. This could result in inefficiencies in liquidity provision, loss of gas fees because due to asset statement it consume all the gas and won't return anything and convertandTransfer will never be able to transfer asset to DAO.

## Code Snippet
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCT/OCT_DAO.sol#L87
https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/lockers/OCT/OCT_YDL.sol#L97
```javascript
assert(IERC20(asset).allowance(address(this), router1INCH_V5) == 0);
```

## Tool used

Manual Review

## Recommendation

Reset Allowances After Use: After interacting with the Uniswap router and completing the liquidity provision, reset the allowances for the pair assets and ZVE tokens to zero to ensure they are not left with excessive allowances.



## Discussion

**pseudonaut**

Valid

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> borderline low/medium, the report has just generic suggestion without specific POC (exact scenario) when this can actually happen. Normally, when doing a swap, full allowance is used to do it. It appears that 1inch's fillOrderRFQ might not use full allowance if there is no matching order in the orderbook, but I'm not 100% sure. Watson should have provided the POC to prove his point.



# Issue M-15: A user can let the last payment of his loan `default` to avoid paying `lateFee and interestOwed` 

Source: https://github.com/sherlock-audit/2024-03-zivoe-judging/issues/643 

The protocol has acknowledged this issue.

## Found by 
Shield
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



## Discussion

**pseudonaut**

Not of concern

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**panprog** commented:
> medium, allows to avoid paying interest and late fees for the last period by letting the loan default and then repaying full amount. This might lead to borrowers skipping the last interest payment by using this strategy, which will harm the protocol users by decreasing the yield.



