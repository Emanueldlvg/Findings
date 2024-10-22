# pause and unpause functionality is not implemented in DevFund, EntityTrading, EntropyGenerator, NukeFund, EntityForging, and TraitForgeNft contracts

## Impact

The related contract cannot be paused or unpaused.

## Proof of Concept

`DevFund`, `EntityTrading`, `EntropyGenerator`, `NukeFund`, `EntityForging`, and `TraitForgeNft` contracts inherit the modifier from the `Pausable.sol` contract. The goal is for the contracts above to be able to apply the `pause` and `unpause` functions to the desired function.

The problem here is, the function to change the `pause` status to `unpause` and vice versa is an internal function so it requires an external function to access it.

```solidity
function _pause() internal virtual whenNotPaused {
        _paused = true;
        emit Paused(_msgSender());
    }
    
function _unpause() internal virtual whenPaused {
        _paused = false;
        emit Unpaused(_msgSender());
    }
```

Since each related contract that inherits `Pausable.sol` contract does not have external `pause` / `unpause` function in it, then the `pause` / `unpause` function cannot be used when needed.

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider add an external function in every related contracts and make sure have access control

```solidity
function pause() external whenNotPaused onlyOwner {
        _paused;
    }
    
function unpause() external whenPaused onlyOwner {
        _unpause;
    }
```
