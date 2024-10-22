# `rOUSG::burn()` always revert if the account is not on the KYC list

## Impact

`rOUSG::burn()` will always revert, `BURNER_ROLE` never got `OUSG` from the account and `rOUSG` amount will be locked in the account.

## Proof of Concept

The scenario below :

1. `BURNER_ROLE` call `burn` function for account which have been removed or subject to sanctions and enter the `rOUSG` value
2. Then for burning, [_burnShares](https://github.com/code-423n4/2024-03-ondo-finance/blob/be2e9ebca6fca460c5b0253970ab280701a15ca1/contracts/ousg/rOUSG.sol#L554-L570) will be called
3. Before making any changes, `_beforeTokenTransfer` will be called. This is to ensure that the relevant account is on the KYC list or is not subject to sanctions. 

```solidity
  function _beforeTokenTransfer(
    address from,
    address to,
    uint256
  ) internal view {
    // Check constraints when `transferFrom` is called to facliitate
    // a transfer between two parties that are not `from` or `to`.
    if (from != msg.sender && to != msg.sender) {
      require(_getKYCStatus(msg.sender), "rOUSG: 'sender' address not KYC'd");
    }

    if (from != address(0)) {
      // If not minting
      require(_getKYCStatus(from), "rOUSG: 'from' address not KYC'd");
    }

    if (to != address(0)) {
      // If not burning
      require(_getKYCStatus(to), "rOUSG: 'to' address not KYC'd");
    }
  }
```

Since the account have been remove from KYC list, `rOUSG::burn()` will always revert. 

Note : 

1. From conversations with the team, this function is intended for certain situations, especially in security matters. If this function is called to forcefully withdraw the `OUSG` from the account (perhaps the account was a problem account at the end and was forcibly removed from the KYC List or was subject to sanctions) by burning the `OUSG` then this cannot be done.
2. Regarding known issues in the README :

> **Sanction or KYC related edge cases - specifically when a user’s `KYCRegistry` or Sanction status changes in between different actions, leaving them at risk of their funds being locked. If someone gets sanctioned on the Chainalysis Sanctions Oracle or removed from Ondo Finance’s KYC Registry their funds are locked.**
> 

This issue is not only locked user funds but it makes `BURNER_ROLE` never get the desired `OUSG` amount and makes this function always revert, so the impact of this issue is more than written from a known issue.

### Coded POC

Copy this test code to `rOUSG.t.sol` and run `npm run test-forge`

```solidity
function testrOUSGAlwaysRevertIfAccountRemovedFromKYCList()
    public
    dealAliceROUSG(1e18) //add alice rOUSG
  {
    //remove alice from KYC List
    vm.prank(OUSG_GUARDIAN);
    _removeAddressFromKYC(OUSG_KYC_REQUIREMENT_GROUP, alice);

    //use GUARDIAN as BURNER ROLE 
    vm.prank(OUSG_GUARDIAN);
    vm.expectRevert("rOUSG: 'from' address not KYC'd");
    rOUSGToken.burn(alice, 100e18);
  }
```

Result 

```solidity
[PASS] testrOUSGAlwaysRevertIfAccountRemovedFromKYCList() (gas: 202125)
```

## Tools Used

Manual review

## Recommended Mitigation Steps

1. If `BURNER_ROLE` very trusted, in `_burnShares` function skip the `_beforeTokenTransfer` part
2. From conversations with the team, they will adding back the account the KYC List → call burn function → remove the account. In my opinion, this not safe. Malicious actor can prepare for some attack and may front-run the transaction
