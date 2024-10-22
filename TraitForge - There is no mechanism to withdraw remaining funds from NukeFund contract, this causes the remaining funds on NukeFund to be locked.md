# There is no mechanism to withdraw remaining funds from NukeFund contract, this causes the remaining funds on NukeFund to be locked

## Impact

The remaining funds on NukeFund contract to be locked

## Proof of Concept

In general, the `NukeFund.sol` contract has a function for users who want to burn the NFTs they own and exchange them for the funds available in the contract. The calculation of funds that users receive is determined from various parameters and the maximum funds that can be claimed are 50% of the total funds available in the contract.

```solidity
uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size
```

This way, there will always be funds remaining from each claim made by the user and cannot be withdrawn by the TraitForge team and are locked forever in the contract. 

This would be a problem if the `NukeFund.sol` contract had a vulnerability and the funds were required to be moved. Of course this cannot be done because the feature does not exist.

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider add withdraw function which can only be done by the owner (preferably a multisig)
