Glamorous Cloud Walrus

medium

# `ZivoeDAO` Incorrect use of `safeDecreaseAllowance` renders allowance safety mechanism ineffective

## Summary

`ZivoeDAO#push()` and `ZivoeDAO#pushMulti()` are implemented to ensure that a `locker` has no remaining allowance once the function concludes. However, the `SafeERC20#safeDecreaseAllowance()` function is used incorrectly which compromises the intended safety mechanism. This flaw breaks an invariant of the protocol and renders the safety mechanism ineffective, potentially allowing a compromised or malicious `locker` to withdraw funds from the DAO.

## Vulnerability Detail

Let's take a look at the `ZivoeDAO#push()` function:

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeDAO.sol#L239-L247

In essence, the function pushes ERC20 token to a `locker` and then ensures that the `locker` has no allowance available. This safeguard is crucial as highlighted by the comment in the code:

```sol
// ZivoeDAO MUST ensure "locker" has 0 allowance before this function concludes.
```

The function is supposed to reset the allowance to zero by calling `IERC20(asset).safeDecreaseAllowance(locker, 0)`. However, due to a misunderstanding of how `SafeERC20#safeDecreaseAllowance()` works, it fails to set the allowance to zero. The function decreases the allowance by the amount specified, meaning a call with `0` does not affect the existing allowance.

This incorrect implementation does not achieve the intended effect of ensuring the `locker` has no remaining allowance. Consequently, if a `locker` retains any allowance post-execution, it can still access DAO funds.
## Impact

The likelihood of exploiting this vulnerability is low, but if it does occur, the consequences could be severe. The intended safety feature in the `ZivoeDAO#push()` function fails to eliminate a `locker`'s allowance due to a coding error. This leaves the DAO vulnerable to significant losses if a compromised `locker` exploits this flaw to access funds.

## Code Snippet

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeDAO.sol#L246

https://github.com/sherlock-audit/2024-03-zivoe/blob/main/zivoe-core-foundry/src/ZivoeDAO.sol#L299

## Tool used

Manual Review

## Recommendation

Use

```sol
IERC20(asset).safeDecreaseAllowance(locker, IERC20(asset).allowance(address(this), locker));
```

instead of

```sol
IERC20(asset).safeDecreaseAllowance(locker, 0);
```

This will set the locker's allowance to zero by decreasing it by the current allowance amount.