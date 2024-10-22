# The ERC-5095 standard is not followed correctly

## Impact

Functionality is not working as expected 

## Proof of Concept

For the `maxRedeem` function, in [EIP-5095](https://eips.ethereum.org/EIPS/eip-5095) it is stated :

> MUST factor in both global and user-specific limits, like if redemption is entirely disabled (even temporarily) it MUST return 0
> 

Spectra has a pause function which can deactivate the `Deposit`, `Redeem` and `Withdraw` functions. The main problem is that when `Redeem` is paused the return value of `maxRedeem` should be `0` but in this case it is not, the return value of `maxRedeem` will still return what it usually does even when the `Redeem` function is disabled. 

This happens because when the user call the `maxRedeem` function it will produce a return value from the `_maxBurnable` function. The code can be seen below :

```solidity
    function maxRedeem(address owner) public view override returns (uint256) {
        return _maxBurnable(owner);

		
		function _maxBurnable(address _user) internal view returns (uint256 maxBurnable) {
        if (block.timestamp >= expiry) {
            maxBurnable = balanceOf(_user);
        } else {
            uint256 ptBalance = balanceOf(_user);
            uint256 ytBalance = IYieldToken(yt).balanceOf(_user);
            maxBurnable = (ptBalance > ytBalance) ? ytBalance : ptBalance;
        }
    }
```

As can be seen in the codebase, this function does not have a modifier or logic that differentiates when the main Redeem function is in pause / unpause state so whatever state the return value is in, it will still return what it usually does. This also applies to the [`maxDeposit`](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L441) function, that function also does not follow existing standards and has the same problem. 

Because `PrincipalToken` must follow EIP-5095 is main invariant, this issue breaks the main invariant and there are not just one but two functions that do not follow the existing standard. It can be concluded :

**Impact:** Medium, as functionality is not working as expected but without a value loss

**Likelihood:** Medium, as multiple methods are not compliant with the standard

## Tools Used

Manual review

## Recommended Mitigation Steps

Go through the [standards](https://eips.ethereum.org/EIPS/eip-5095) and follow it all.
