# Fee on Transfer Token can't be used for budget asset 

## Summary

A summary with the following structure: **{root cause} will cause [a/an] {impact} for {affected party} as {actor} will {attack path}**

Based on README :

> **If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of [weird tokens](https://github.com/d-xo/weird-erc20) you want to integrate?**
> 
> 
> Protocol should work with all native and ERC20 tokens that adhear to standard including weird tokens.
> 
> No assumptions are made in regards to the properties of integrating tokens and weird trait tokens should be handled normally. Specifically we'd expect to interact with USDT and other stable coins/decimal 6 coins and erc20z from Zora as well as other fractional NFT contracts as long as they adhere to ERC20 standard. We aren't whitelisting tokens so any erc20 should work (including weird ones).
> 

Thus the protocol assumes `FoT` tokens (e.g. `STA`, `PAXG`) can be used as budget assets but in reality they are not.

## **Root Cause**

*In [ManagedBudget:66-73](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-Emanueldlvg/blob/78930f2ed6570f30e356b5529bd4bcbe5194eb8b/boost-protocol/packages/evm/contracts/budgets/ManagedBudget.sol#L66-L73) not compatible with `FoT` Token*

Fee on Transfer Token can't be used for budget asset 

## **PoC**

```solidity
        } else if (request.assetType == AssetType.ERC20) {
            FungiblePayload memory payload = abi.decode(request.data, (FungiblePayload));

            // Transfer `payload.amount` of the token to this contract
            request.asset.safeTransferFrom(request.target, address(this), payload.amount);
            if (request.asset.balanceOf(address(this)) < payload.amount) {
                revert InvalidAllocation(request.asset, payload.amount);
            }
```

If allocator calls the `allocate()` function with `FoT` token as asset, then the function will always revert because the balance at address(this) < the amount transferred 

## **Mitigation**

Consider adding some logic to handle `FoT` token as budget asset
