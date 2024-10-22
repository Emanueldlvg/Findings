# The ERC-4626 standard is not followed correctly

## Impact

Functionality is not working as expected 

## Proof of Concept

1. `deposit`, `mint`, `withdraw`, and `redeem` functions

As per [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) :

> If implementors intend to support EOA account access directly, they should consider adding an additional function call for `deposit`/`mint`/`withdraw`/`redeem` with the means to accommodate slippage loss or unexpected deposit/withdrawal limits, since they have no other means to revert the transaction if the exact output amount is not achieved.
> 

But in this codebase, the protocol does not follow this. There is no slippage protection for these four main functions which results in users receiving shares (`deposit`) less than they should and users receiving assets (`withdraw`) less than they should. Lets take a look these function in the codebase :

```solidity
    function deposit(uint256 assets, address receiver) external override returns (uint256) {
        (, uint256 shares) = _deposit(receiver, assets, false, "");
        return shares;
    }

    /// @inheritdoc IERC4626
    function mint(uint256 shares, address receiver) external override returns (uint256) {
        (uint256 assets,) = _deposit(receiver, shares, true, "");
        return assets;
    }

    /// @inheritdoc IERC4626
    function withdraw(uint256 assets, address receiver, address owner) external override returns (uint256) {
        (, uint256 shares) = _withdraw(receiver, owner, assets, false);
        return shares;
    }

    /// @inheritdoc IERC4626
    function redeem(uint256 shares, address receiver, address owner) external override returns (uint256) {
        (uint256 assets,) = _withdraw(receiver, owner, shares, true);
        return assets;
    }
```

It can be seen from the codebase, when the user calls these four functions they will be passed to the internal `_deposit` and `_withdraw` functions, the difference is the `isShares` variable. Lets take a look those functions :

```solidity
    function _deposit(address receiver, uint256 amount, bool isShare, bytes memory permitData)
        internal
        returns (uint256 assets, uint256 shares)
    {
        (, uint256 newLendExchangeRateX96) = _updateGlobalInterest();

        _resetDailyLendIncreaseLimit(newLendExchangeRateX96, false);

        if (isShare) {
            shares = amount;
            assets = _convertToAssets(shares, newLendExchangeRateX96, Math.Rounding.Up);
        } else {
            assets = amount;
            shares = _convertToShares(assets, newLendExchangeRateX96, Math.Rounding.Down);
        }

        if (permitData.length > 0) {
            (ISignatureTransfer.PermitTransferFrom memory permit, bytes memory signature) =
                abi.decode(permitData, (ISignatureTransfer.PermitTransferFrom, bytes));
            permit2.permitTransferFrom(
                permit, ISignatureTransfer.SignatureTransferDetails(address(this), assets), msg.sender, signature
            );
        } else {
            // fails if not enough token approved
            SafeERC20.safeTransferFrom(IERC20(asset), msg.sender, address(this), assets);
        }

        _mint(receiver, shares);

        if (totalSupply() > globalLendLimit) {
            revert GlobalLendLimit();
        }

        if (assets > dailyLendIncreaseLimitLeft) {
            revert DailyLendIncreaseLimit();
        } else {
            dailyLendIncreaseLimitLeft -= assets;
        }

        emit Deposit(msg.sender, receiver, assets, shares);
    }

    // withdraws lent tokens. can be denominated in token or share amount
    function _withdraw(address receiver, address owner, uint256 amount, bool isShare)
        internal
        returns (uint256 assets, uint256 shares)
    {
        (uint256 newDebtExchangeRateX96, uint256 newLendExchangeRateX96) = _updateGlobalInterest();

        if (isShare) {
            shares = amount;
            assets = _convertToAssets(amount, newLendExchangeRateX96, Math.Rounding.Down);
        } else {
            assets = amount;
            shares = _convertToShares(amount, newLendExchangeRateX96, Math.Rounding.Up);
        }

        // if caller has allowance for owners shares - may call withdraw
        if (msg.sender != owner) {
            _spendAllowance(owner, msg.sender, shares);
        }

        (, uint256 available,) = _getAvailableBalance(newDebtExchangeRateX96, newLendExchangeRateX96);
        if (available < assets) {
            revert InsufficientLiquidity();
        }

        // fails if not enough shares
        _burn(owner, shares);
        SafeERC20.safeTransfer(IERC20(asset), receiver, assets);

        // when amounts are withdrawn - they may be deposited again
        dailyLendIncreaseLimitLeft += assets;

        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }
```

In the `_deposit` function, the protocol only takes into account the user's upper deposit limit with the `DailyLendIncreaseLimit` and `GlobalLendLimit` variables but does not take into account how many shares the user should receive when making a deposit. So when a user makes a deposit, he may receive shares less than the amount of the deposit he made. 

Meanwhile, for the `_withdraw` function, the protocol only takes into account the amount of liquidity available when the user makes a withdrawal but does not take into account whether the assets received by the user correspond to the shares burned. So when a user makes a withdrawal, the user may receive assets less than they should be and burning more shares than necessary. 

2. `maxDeposit` and `maxMint` functions

As per [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) :

> MUST factor in both global and user-specific limits, like if deposits are entirely disabled (even temporarily) it MUST return 0.
> 

But in the codebase, the protocol does not follow this. When the user calls these two functions, in certain circumstances the results of these two functions do not match what the user can do. Lets take a look these function in the codebase :

```solidity
    /// @inheritdoc IERC4626
    function maxDeposit(address) external view override returns (uint256) {
        (, uint256 lendExchangeRateX96) = _calculateGlobalInterest();
        uint256 value = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up);
        if (value >= globalLendLimit) {
            return 0;
        } else {
            return globalLendLimit - value;
        }
    }

    /// @inheritdoc IERC4626
    function maxMint(address) external view override returns (uint256) {
        (, uint256 lendExchangeRateX96) = _calculateGlobalInterest();
        uint256 value = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up);
        if (value >= globalLendLimit) {
            return 0;
        } else {
            return _convertToShares(globalLendLimit - value, lendExchangeRateX96, Math.Rounding.Down);
        }
    }
```

As seen in the functions above, the protocol only takes into account the upper limit using `globalLendLimit`, whereas in the codebase there is also a `dailyLendIncreaseLimitLeft` variable which functions to limit the number of deposits / mints made each day. When the user's `dailyLendIncreaseLimitLeft` value reaches the maximum value, the user cannot make deposits or mints. Thus, the `maxDeposit` and `maxMint` functions should have a return value = 0 because deposit and mint are in the disabled state.

## Tools Used

Manual review

## Recommended Mitigation Steps

1. Go through [the standard](https://eips.ethereum.org/EIPS/eip-4626) and follow it all.
2. Add slippage protection 

```solidity
    function deposit(uint256 assets, address receiver, uint256 minShares) external override returns (uint256) {
        (, uint256 shares) = _deposit(receiver, assets, false, "");
        if (shares < minShares) {
	        revert slippageProtection }
        return shares;
    }

    /// @inheritdoc IERC4626
    function mint(uint256 shares, address receiver, uint256 minAssets) external override returns (uint256) {
        (uint256 assets,) = _deposit(receiver, shares, true, "");
        if (assets < minAssets) {
	        revert slippageProtection }
        return assets;
    }

    /// @inheritdoc IERC4626
    function withdraw(uint256 assets, address receiver, address owner, uint256 maxShares) external override returns (uint256) {
        (, uint256 shares) = _withdraw(receiver, owner, assets, false);
        if (shares > maxShares) {
	        revert slippageProtection }
        return shares;
    }

    /// @inheritdoc IERC4626
    function redeem(uint256 shares, address receiver, address owner, uint256 mintAssets) external override returns (uint256) {
        (uint256 assets,) = _withdraw(receiver, owner, shares, true);
        if (assets < minAssets) {
	        revert slippageProtection }
        return assets;
    }
```

3. Also pay attention to user specific variables

```solidity
    /// @inheritdoc IERC4626
    function maxDeposit(address) external view override returns (uint256) {
        (, uint256 lendExchangeRateX96) = _calculateGlobalInterest();
        uint256 value = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up);
        if (value >= globalLendLimit || value > dailyLendIcreaseLimitLeft) {
            return 0;
        } else {
            return globalLendLimit - value;
        }
    }

    /// @inheritdoc IERC4626
    function maxMint(address) external view override returns (uint256) {
        (, uint256 lendExchangeRateX96) = _calculateGlobalInterest();
        uint256 value = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up);
        if (value >= globalLendLimit || value > dailyLendIcreaseLimitLeft) {
            return 0;
        } else {
            return _convertToShares(globalLendLimit - value, lendExchangeRateX96, Math.Rounding.Down);
        }
    }
```
