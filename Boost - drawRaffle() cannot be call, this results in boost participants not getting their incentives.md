# drawRaffle() cannot be call, this results in boost participants not getting their incentives

## Summary

A summary with the following structure: **{root cause} will cause [a/an] {impact} for {affected party} as {actor} will {attack path}**

When a boost creator create a boost, incentives are also created. There are 2 types of strategies for `ERC20Incentive`, the first is `POOL` and the second is `RAFFLE`. Problems arise when boost creator use `RAFFLE` type incentives, this is because the `claim()` function in `ERC20Incentive` only enter the address of boost participants into the array that will be drawn when `drawRaffle()` is called. While `drawRaffle()` itself cannot be called because of the `onlyOwner` modifier where the boost core contract as the owner does not have a function to call `drawRaffle()` on `ERC20Incentive`. This causes boost participants to never get their incentives because the raffle winners cannot be drawn. 

NOTE
This issue is not about the boost creator or owner behaving maliciously to prevent boost participants from claiming their rewards but about the broken functionality of `ERC20Incentive.strategy:RAFFLE`

## **Root Cause**

*In [BoostCore.sol](https://github.com/sherlock-audit/2024-06-boost-aa-wallet-Emanueldlvg/blob/78930f2ed6570f30e356b5529bd4bcbe5194eb8b/boost-protocol/packages/evm/contracts/BoostCore.sol#L25) there is no function to call `drawRaffle()` function on `ERC20Incentive`*

## **Impact**

1. Broken functionality when boost uses `ERC20Incentive` of the `RAFFLE` type as incentives for boost participants
2. Boost participants not getting their incentives 

## **PoC**

1. Boost creator create a boost by calling the `createBoost()` function
2. Then incentives are created with the `_makeIncentives()` function, where the incentive initiation process occurs
3. Next to incentives initiation process by calling the `initialize()` function in `ERC20Incentive` contract 

```solidity
function initialize(bytes calldata data_) public override initializer {
        InitPayload memory init_ = abi.decode(data_, (InitPayload));

        if (init_.reward == 0 || init_.limit == 0) revert BoostError.InvalidInitialization();

        // Ensure the maximum reward amount has been allocated
        uint256 maxTotalReward = init_.strategy != Strategy.RAFFLE ? init_.reward * init_.limit : init_.reward;
        uint256 available = init_.asset.balanceOf(address(this));
        if (available < maxTotalReward) {
            revert BoostError.InsufficientFunds(init_.asset, available, maxTotalReward);
        }

        asset = init_.asset;
        strategy = init_.strategy;
        reward = init_.reward;
        limit = init_.limit;
        _initializeOwner(msg.sender);
    }
```

there is `_initializeOwner(msg.sender)`, which means the `BoostCore` is the owner of the `ERC20Incentive` contract 

4. Boost participants call `claimIncentive()` , but it can be seen in the `claim()` function in `ERC20Incentive` that it only enters the boost participants address into the entries array if `Strategy.RAFFLE`

```solidity
    function claim(address claimTarget, bytes calldata) external override onlyOwner returns (bool) {
        if (!_isClaimable(claimTarget)) revert NotClaimable();

        if (strategy == Strategy.POOL) {
            claims++;
            claimed[claimTarget] = true;

            asset.safeTransfer(claimTarget, reward);

            emit Claimed(claimTarget, abi.encodePacked(asset, claimTarget, reward));
            return true;
        } else {
            claims++;
            claimed[claimTarget] = true;
            entries.push(claimTarget);

            emit Entry(claimTarget);
            return true;
        }
    }
```

5. Then to find out the winner of `RAFFLE` and send the prize as an incentive, the `drawRaffle()` function needs to be called. But because `BoostCore` as the owner of `ERC20Incentive` does not have a function to call `drawRaffle()`, the `drawRaffle()` function cannot be called and the winner is never known and the incentive is never sent

```solidity
    function drawRaffle() external override onlyOwner {
```

## **Mitigation**

Consider add function to call `drawRaffle()` on `BoostCore` with modifier (for boost owner or admin roles)
