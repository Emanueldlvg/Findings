# curvesTokenSubject may set referralFeeDestination to malicious address that cause gas griefing in _transferFees function and DoS the protocol

## Impact

Lost all fees and causes DoS on the protocol

## Proof of Concept

`curvesTokenSubject` can input `referralFeeDestination` with any address he wants. This can happen with this function:

```solidity
File : contracts/Curves.sol

function setReferralFeeDestination(
    address curvesTokenSubject,
    address referralFeeDestination_
    ) public onlyTokenSubject(curvesTokenSubject) {
    referralFeeDestination[curvesTokenSubject] = referralFeeDestination_;
}
```

Then when `curvesTokenSubject` makes a sale or purchase transaction, the fee will be transferred according to the address and in proportion to this function:

```solidity
File : contracts/Curves.sol

function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {
        (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holderFee, ) = getFees(price);
        {
            bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);
            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
            }
            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }
            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }

            if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {
                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }
        }
        emit Trade(
            msg.sender,
            curvesTokenSubject,
            isBuy,
            amount,
            price,
            protocolFee,
            subjectFee,
            isBuy ? supply + amount : supply - amount
        );
    }
```

The focus here is the low-level `call` made to the `referralFeeDestination` address :

```solidity
{
  (bool success3, ) = referralDefined 
       ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
       : (true, bytes(""));
   if (!success3) revert CannotSendFunds();
}
```

`(bool success, ) = (bool success, bytes memory data)` it means even though the data is omitted it doesn’t mean that the contract does not handle it. The `bytes data` return from `receiver` will be copied to memory. Memory allocation becomes very costly if the payload is big, so this means that if a `receiver` implements a fallback function that returns a huge payload, then the `msg.sender` of the transaction, in our case the `Curves` contract, will have to pay a huge amount of gas for copying this payload to memory.

## Tools Used

Manual review

## Recommended Mitigation Steps

Use a low-level assembly call because it does not automatically copy the return bytes of data :

```solidity
bool success;
assembly {
    success := call (
		gasLimit, // gas limit
		receiver, // receiver
		amount, // amount
		0, // in
		0, // insize
		0, // out
		0 // outsize
    ) 
}
```
