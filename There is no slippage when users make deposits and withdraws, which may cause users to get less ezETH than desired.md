# There is no slippage when users make deposits and withdraws, which may cause users to get less ezETH than desired

## Impact

Users may get less `ezETH` / collateral token than desired

## Proof of Concept

[RestakeManager::deposit](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L491-L576) & [RestakeManager::depositETH](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L592-L616) functions for users to make deposits in the form of collateral tokens (`stETH`, `wbETH`) and `ETH`. 

When a user makes a deposit with collateral tokens, the value of the number of tokens is checked with [renzoOracle::lookupTokenValue](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L71-L81). After getting the value, the amount of `ezETH` that will be requested from the user is determined with [renzoOracle::calculateMintAmount](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L123-L149).

```solidity
        if (mintAmount == 0) revert InvalidTokenAmount();
```

The problem arises because the `calculateMintAmount()` sanity check function is carried out only if `mintAmount = 0` then the function will revert. Then in the `deposit()` function there is no checking whether the `ezETHtoMint` value (the result of calling the `calculateMintAmount()` function) matches what is desired. 

NOTE

This also applies if the user withdraws or redeems the `ezETH` token they have in order to get `ETH` or other collateral tokens

Instances :

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206-L263

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider adding check whether the `ezETH` that will be requested for the user matches what is desired

```solidity
    function deposit(
        IERC20 _collateralToken,
        uint256 _amount,
        uint256 _referralId,
        uint256 minezETH
    ) public nonReentrant notPaused {
    
      uint256 ezETHToMint = renzoOracle.calculateMintAmount(
      totalTVL,
      collateralTokenValue,
      ezETH.totalSupply()
    );
    
    if (ezETHtoMint < minezETH) {
	    revert lessezETH;
	    }
	  }
	    
	    
    function depositETH(
		    uint256 _referralId,
		    uint256 minezETH
   ) public payable nonReentrant notPaused {
   
      uint256 ezETHToMint = renzoOracle.calculateMintAmount(
      totalTVL,
      msg.value,
      ezETH.totalSupply()
    );
    
    if (ezETHtoMint < minezETH) {
	    revert lessezETH;
	    }
	 }
```
