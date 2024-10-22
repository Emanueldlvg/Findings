# Malicious actors can make deposits to other user DNFT ID to continue to block that user from making withdrawal

## Impact

User who own the deposited ID cannot make withdrawal

## Proof of Concept

[VaultManagerV2::deposit()](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L119-L131) functions for users to deposit collateral into the vault they own. This function allows anyone to make a deposit to a valid DNFT ID using the `isValidDNft` modifier. In this function there is also logic to block the possibility of a flash loan attack by applying the code as below:

```solidity
    idToBlockOfLastDeposit[id] = block.number;
```

By implementing this, users cannot make withdrawals on the same block number

```solidity
    if (idToBlockOfLastDeposit[id] == block.number) revert DepositedInSameBlock();
```

The main problem here lies in the modifier used, [isValidDNft](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L42-L44). `isValidDNft` only checks that the owner of the ID is not address(0). This way, it allows anyone to make a deposit to the vault from a valid ID and blocks the owner of the vault (deposited ID) from making withdrawals.

The scenario will be like this :

1. Bob as malicious actor deposit to vault owned by Alice by entering the address of the vault and DNFT ID owned by Alice.
2. This will work because `isValidDNft` only checks whether Alice's DNFT ID = address(0) or not (since the beginning Alice's DNFT ID is valid, not address(0))
3. Once successful, the value of `idToBlockOfLastDeposit[id]` will be updated according to the `block.number` when Bob made the deposit.
4. That way, when Alice wants to make a deposit on the same `block.number`, Alice's withdrawal will always revert because `idToBlockOfLastDeposit[id] == block.number`
5. Bob can do this continuously because there is no minimum amount requested for a user to make a deposit, he can even send `1 wei` and update the value of `idToBlockOfLastDeposit`

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider limiting only DNFT ID owners who can make deposits using the `isDNftOwner` modifier

```solidity
  function deposit(
    uint    id,
    address vault,
    uint    amount
  ) 
    external 
      isValidDNft(id)
      isDNftOwner(id)
  {
    idToBlockOfLastDeposit[id] = block.number;
    Vault _vault = Vault(vault);
    _vault.asset().safeTransferFrom(msg.sender, address(vault), amount);
    _vault.deposit(id, amount);
  }
```
