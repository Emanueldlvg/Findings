# The delegatecall function does not change the storage in the target contract so the user cannot set the address of a user who can manage SAFE with the allowSafe function

## Impact

User can’t set other address for manage SAFE

## Proof of Concept

![Open Dollar.jpg](https://prod-files-secure.s3.us-west-2.amazonaws.com/93fed454-f35a-4808-afcf-14eb7aa5f207/ce72cdd9-9046-4ab7-b83d-f2f887caff69/Open_Dollar.jpg)

User can only interact with SAFE using `ODProxy`. The user calls the `execute` function on `ODProxy` to make a `delegatecall` to the target contract to execute the intended function. But the problem here is that the `delegatecall` function cannot change the storage of the target contract. In this case, user use `execute` function to make `delegatecall` to execute `allowsafe` function on `ODSafeManager`with the aim of setting another address that can manage SAFE but because `delegatecall` does not change the storage in the target contract, this will be in vain and the `allowSafe` function will not work.

## Tools Used

Manual review

## Recommended Mitigation Steps

Consider making a direct call to the `allowSafe` function from `vault721.sol` contract
