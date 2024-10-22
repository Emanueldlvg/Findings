# Temple Gold token will lost if user using account abstraction wallet 

## Summary

Users with account abstraction wallets have a different address across different chains for same account, if users using an account abstraction wallet bridge the asset, assets will be moved to wrong address and lost permanently.

## Vulnerability Details

```solidity
if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }
```

In the [`TempleGold::send()`](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/TempleGold.sol#L290) , the user must enter the `to` parameter equal to `msg.sender` which means, the address of `msg.sender` and `to` must be the same otherwise the function will revert. 

As explained above, if the user uses an abstraction wallet account then the address on the destination chain is different from the origin chain then the user will lose the Temple Gold token he sent.

## Impact

User will lose the Temple Gold token he sent using the `send()` function

## Tools Used

Manual Review

## Recommended Mitigation

Pass in the warning for account abstraction wallet holders to not send Temple Gold token with `send()` function.
