# TokenManager::_transfer() not compatible with fee on transfer and rebasing token

## Summary

`TokenManager::_transfer()` not compatible with fee on transfer and rebasing token

## Vulnerability Details

```solidity

        if (fromBalanceAft != fromBalanceBef - _amount) {
            revert TransferFailed();
        }

        if (toBalanceAft != toBalanceBef + _amount) {
            revert TransferFailed();
        }
    }
```

As can be seen in the code above, in the `_transfer` function there is a check on the balance of tokens before and after for the `from` and `to` addresses. This does not work with fee on transfer tokens because as the name suggests, when transferring tokens it requires a fee. In addition, this mechanism does not work with rebasing tokens because the balance changes.

## Impact

`_transfer` function will always revert

## Tools Used

Manual Review

## Recommended Mitigation

Make sure fee on transfer and rebasing tokens didn’t include in whitelist token
