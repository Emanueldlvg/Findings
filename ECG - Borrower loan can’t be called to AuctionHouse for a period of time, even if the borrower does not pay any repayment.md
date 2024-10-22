# Borrower loan can’t be called to AuctionHouse for a period of time, even if the borrower does not pay any repayment
## Impact

Loan can’t be called to `AuctionHouse` even if borrower does not pay any repayment until term is offboard

## Proof of Concept

There are two conditions for a loan can be called, when the term is offboard and the value of `partialRepayDelayPassed` is true. The code is in [`LendingTerm::_call`](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/2376d9af792584e3d15ec9c32578daa33bb56b43/src/loan/LendingTerm.sol#L652-L656) : 

```solidity
File : src/loan/LendingTerm.sol	
	
     require(
        GuildToken(refs.guildToken).isDeprecatedGauge(address(this)) ||
             partialRepayDelayPassed(loanId),
        "LendingTerm: cannot call"
    );
```

The main problem is with `partialRepayDelayPassed`, if term set `maxDelayBetweenPartialRepay == 0`, it will make return value of this function always `false`. The code : 

```solidity
File : src/loan/LendingTerm.sol	

if (params.maxDelayBetweenPartialRepay == 0) return false;
```

This will mean that loan borrowers cannot be called to `AuctionHouse` even if they do not pay any repayment until the term is offboarded. The worst thing is, if the term is not offboarded then the borrower has a free loan and its collateral is safe. Apart from that, this also harms lenders in term because it reduces the amount of interest earned from interest payments made by borrowers.

**POC (Test)**

For this test, the same environment and setup as in `lendingTerm.t.sol` is used and only the `maxDelayBetweenPartialRepay` parameter is changed to `0`. This test shows that the loan cannot be called unless the term has been offboarded even borrower didn’t make any repayment.

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.13;

import {Clones} from "@openzeppelin/contracts/proxy/Clones.sol";

import {Test} from "@forge-std/Test.sol";
import {Core} from "@src/core/Core.sol";
import {CoreRoles} from "@src/core/CoreRoles.sol";
import {MockERC20} from "@test/mock/MockERC20.sol";
import {SimplePSM} from "@src/loan/SimplePSM.sol";
import {GuildToken} from "@src/tokens/GuildToken.sol";
import {CreditToken} from "@src/tokens/CreditToken.sol";
import {LendingTerm} from "@src/loan/LendingTerm.sol";
import {AuctionHouse} from "@src/loan/AuctionHouse.sol";
import {ProfitManager} from "@src/governance/ProfitManager.sol";
import {RateLimitedMinter} from "@src/rate-limits/RateLimitedMinter.sol";

contract TestMe is Test {
    address private governor = address(1);
    address private guardian = address(2);
    Core private core;
    ProfitManager private profitManager;
    CreditToken credit;
    GuildToken guild;
    MockERC20 collateral;
    SimplePSM private psm;
    RateLimitedMinter rlcm;
    AuctionHouse auctionHouse;
    LendingTerm term;

    // LendingTerm params
    uint256 constant _CREDIT_PER_COLLATERAL_TOKEN = 2000e18; // 2000, same decimals
    uint256 constant _INTEREST_RATE = 0.10e18; // 10% APR
    uint256 constant _MAX_DELAY_BETWEEN_PARTIAL_REPAY = 0; // 2 years
    uint256 constant _MIN_PARTIAL_REPAY_PERCENT = 0.2e18; // 20%
    uint256 constant _HARDCAP = 20_000_000e18;

    uint256 public issuance = 0;

    function setUp() public {
        vm.warp(1679067867);
        vm.roll(16848497);
        core = new Core();

        profitManager = new ProfitManager(address(core));
        collateral = new MockERC20();
        credit = new CreditToken(address(core), "name", "symbol");
        guild = new GuildToken(
            address(core),
            address(profitManager)
        );
        rlcm = new RateLimitedMinter(
            address(core) /*_core*/,
            address(credit) /*_token*/,
            CoreRoles.RATE_LIMITED_CREDIT_MINTER /*_role*/,
            type(uint256).max /*_maxRateLimitPerSecond*/,
            type(uint128).max /*_rateLimitPerSecond*/,
            type(uint128).max /*_bufferCap*/
        );
        auctionHouse = new AuctionHouse(address(core), 650, 1800);
        term = LendingTerm(Clones.clone(address(new LendingTerm())));
        term.initialize(
            address(core),
            LendingTerm.LendingTermReferences({
                profitManager: address(profitManager),
                guildToken: address(guild),
                auctionHouse: address(auctionHouse),
                creditMinter: address(rlcm),
                creditToken: address(credit)
            }),
            LendingTerm.LendingTermParams({
                collateralToken: address(collateral),
                maxDebtPerCollateralToken: _CREDIT_PER_COLLATERAL_TOKEN,
                interestRate: _INTEREST_RATE,
                maxDelayBetweenPartialRepay: _MAX_DELAY_BETWEEN_PARTIAL_REPAY,
                minPartialRepayPercent: _MIN_PARTIAL_REPAY_PERCENT,
                openingFee: 0,
                hardCap: _HARDCAP
            })
        );
        psm = new SimplePSM(
            address(core),
            address(profitManager),
            address(credit),
            address(collateral)
        );
        profitManager.initializeReferences(address(credit), address(guild), address(psm));

        // roles
        core.grantRole(CoreRoles.GOVERNOR, governor);
        core.grantRole(CoreRoles.GUARDIAN, guardian);
        core.grantRole(CoreRoles.CREDIT_MINTER, address(this));
        core.grantRole(CoreRoles.GUILD_MINTER, address(this));
        core.grantRole(CoreRoles.GAUGE_ADD, address(this));
        core.grantRole(CoreRoles.GAUGE_REMOVE, address(this));
        core.grantRole(CoreRoles.GAUGE_PARAMETERS, address(this));
        core.grantRole(CoreRoles.CREDIT_MINTER, address(rlcm));
        core.grantRole(CoreRoles.RATE_LIMITED_CREDIT_MINTER, address(term));
        core.grantRole(CoreRoles.GAUGE_PNL_NOTIFIER, address(term));
        core.renounceRole(CoreRoles.GOVERNOR, address(this));

        // add gauge and vote for it
        guild.setMaxGauges(10);
        guild.addGauge(1, address(term));
        guild.mint(address(this), _HARDCAP * 2);
        guild.incrementGauge(address(term), _HARDCAP);

        // labels
        vm.label(address(core), "core");
        vm.label(address(profitManager), "profitManager");
        vm.label(address(collateral), "collateral");
        vm.label(address(credit), "credit");
        vm.label(address(guild), "guild");
        vm.label(address(rlcm), "rlcm");
        vm.label(address(auctionHouse), "auctionHouse");
        vm.label(address(term), "term");
        vm.label(address(this), "test");
    }

    function testCallMaxDelayBetweenPartialRepay0() public {
        // prepare
        uint256 borrowAmount = 20_000e18;
        uint256 collateralAmount = 15e18;
        collateral.mint(address(this), collateralAmount);
        collateral.approve(address(term), collateralAmount);
        bytes32 loanId = term.borrow(borrowAmount, collateralAmount);

        // check the return value
        assertEq(term.partialRepayDelayPassed(loanId), false);
        vm.warp(block.timestamp + term.YEAR());
        vm.roll(block.number + 1);
        assertEq(term.partialRepayDelayPassed(loanId), false);

        // call
        vm.expectRevert("LendingTerm: cannot call");
        term.call(loanId);
    }
}
```

Result :

```solidity
[PASS] testCallMaxDelayBetweenPartialRepay0() (gas: 365668)
Traces:
  [22745622] test::setUp()
    ├─ [0] VM::warp(1679067867 [1.679e9])
    │   └─ ← ()
    ├─ [0] VM::roll(16848497 [1.684e7])
    │   └─ ← ()
    ├─ [1068723] → new core@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   ├─ emit RoleGranted(role: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleAdminChanged(role: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x55435dd261a4b9b3364963f7738a7a662ad9c84396d64be3365284bb7f0a5041, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x894017c09ca64151bdcdea52a1f0165fafbd24d1a299ee68541fc9b08b334dd1, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xe1c4c7c8669e29e308967879c7054b1e9477115fb5fee6072d66b6c291b29545, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x173bbb6144d15c2757be997bfcbee5bb3b464db18b52cd72d84565f6e77a3eec, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xf213c52f17fbe7c12e60dedf809e77244577de25e041307d406900a545756831, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xc08de0c276688e3ad0194e22b5c58800b27164e8f85cd41c4727cc504f0f1df2, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x08e5a6ba998e059c55ada17d321c304b7ab7f9cbbe1bf0b6b0821bc47635a33e, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x2434c62dad7115c02a27c0967b7aef60b4a026459092461dcb99efc53094cc7e, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xfad0c4e9b1a132544a7dc1dd175e2ff81cd3b28b49f054696650a5f212141ae4, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x094d39d9a418bad7e06fc4c028b3583533bf46288cd20c5c00271ecebc746ece, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0x49755bf9462343aed5d2d014ee520b04d27fe527b1c8dbaba6b741f409ffc3b6, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xd936ee5ca12d3c82cf1b8830fb26bdd319e2e069c44fc3e2c6b563bf2dd631d4, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xb09aa5aeb3702cfd50b6b62bc4532604938f21248a27a1d5ca736082b6819cc1, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xd8aa0f3194971a2a116679f7c2090f6939c8d4e01a2a8d7e41d55e5351469e63, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   ├─ emit RoleAdminChanged(role: 0xfd643c72710c63c0180259aba6b2d05451e3591a24e58b62239378085726f783, previousAdminRole: 0x0000000000000000000000000000000000000000000000000000000000000000, newAdminRole: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55)
    │   └─ ← 2828 bytes of code
    ├─ [2288239] → new profitManager@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   ├─ emit MinBorrowUpdate(when: 1679067867 [1.679e9], newValue: 100000000000000000000 [1e20])
    │   └─ ← 10979 bytes of code
    ├─ [1199553] → new collateral@0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
    │   └─ ← 5649 bytes of code
    ├─ [4423427] → new credit@0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9
    │   └─ ← 21298 bytes of code
    ├─ [4612757] → new guild@0xc7183455a4C133Ae270771860664b6B7ec320bB1
    │   └─ ← 22582 bytes of code
    ├─ [1195464] → new rlcm@0xa0Cb889707d426A7A386870A03bc70d1b0697598
    │   ├─ emit BufferCapUpdate(oldBufferCap: 0, newBufferCap: 340282366920938463463374607431768211455 [3.402e38])
    │   ├─ emit RateLimitPerSecondUpdate(oldRateLimitPerSecond: 0, newRateLimitPerSecond: 340282366920938463463374607431768211455 [3.402e38])
    │   └─ ← 5610 bytes of code
    ├─ [1258248] → new auctionHouse@0x1d1499e622D69689cdf9004d05Ec547d650Ff211
    │   └─ ← 6172 bytes of code
    ├─ [3310387] → new LendingTerm@0xA4AD4f68d0b91CFD19687c881e50f3A00242828c
    │   └─ ← 16423 bytes of code
    ├─ [9031] → new term@0x03A6a84cD762D9707A21605b548aaaB891562aAb
    │   └─ ← 45 bytes of code
    ├─ [251393] term::initialize(core: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], LendingTermReferences({ profitManager: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, guildToken: 0xc7183455a4C133Ae270771860664b6B7ec320bB1, auctionHouse: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211, creditMinter: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, creditToken: 0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9 }), LendingTermParams({ collateralToken: 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a, maxDebtPerCollateralToken: 2000000000000000000000 [2e21], interestRate: 100000000000000000 [1e17], maxDelayBetweenPartialRepay: 0, minPartialRepayPercent: 200000000000000000 [2e17], openingFee: 0, hardCap: 20000000000000000000000000 [2e25] }))
    │   ├─ [251152] LendingTerm::initialize(core: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], LendingTermReferences({ profitManager: 0x2e234DAe75C793f67A35089C9d99245E1C58470b, guildToken: 0xc7183455a4C133Ae270771860664b6B7ec320bB1, auctionHouse: 0x1d1499e622D69689cdf9004d05Ec547d650Ff211, creditMinter: 0xa0Cb889707d426A7A386870A03bc70d1b0697598, creditToken: 0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9 }), LendingTermParams({ collateralToken: 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a, maxDebtPerCollateralToken: 2000000000000000000000 [2e21], interestRate: 100000000000000000 [1e17], maxDelayBetweenPartialRepay: 0, minPartialRepayPercent: 200000000000000000 [2e17], openingFee: 0, hardCap: 20000000000000000000000000 [2e25] })) [delegatecall]
    │   │   ├─ emit CoreUpdate(oldCore: 0x0000000000000000000000000000000000000000, newCore: core: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [1296437] → new SimplePSM@0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF
    │   ├─ [312] collateral::decimals() [staticcall]
    │   │   └─ ← 18
    │   └─ ← 6356 bytes of code
    ├─ [68654] profitManager::initializeReferences(credit: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], guild: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], SimplePSM: [0xD6BbDE9174b1CdAa358d2Cf4D57D1a9F7178FBfF])
    │   ├─ [643] core::hasRole(0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   └─ ← true
    │   └─ ← ()
    ├─ [70477] core::grantRole(0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, 0x0000000000000000000000000000000000000001)
    │   ├─ emit RoleGranted(role: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, account: 0x0000000000000000000000000000000000000001, sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0x55435dd261a4b9b3364963f7738a7a662ad9c84396d64be3365284bb7f0a5041, 0x0000000000000000000000000000000000000002)
    │   ├─ emit RoleGranted(role: 0x55435dd261a4b9b3364963f7738a7a662ad9c84396d64be3365284bb7f0a5041, account: 0x0000000000000000000000000000000000000002, sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0xe1c4c7c8669e29e308967879c7054b1e9477115fb5fee6072d66b6c291b29545, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0xe1c4c7c8669e29e308967879c7054b1e9477115fb5fee6072d66b6c291b29545, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0xf213c52f17fbe7c12e60dedf809e77244577de25e041307d406900a545756831, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0xf213c52f17fbe7c12e60dedf809e77244577de25e041307d406900a545756831, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0xc08de0c276688e3ad0194e22b5c58800b27164e8f85cd41c4727cc504f0f1df2, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0xc08de0c276688e3ad0194e22b5c58800b27164e8f85cd41c4727cc504f0f1df2, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0x08e5a6ba998e059c55ada17d321c304b7ab7f9cbbe1bf0b6b0821bc47635a33e, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x08e5a6ba998e059c55ada17d321c304b7ab7f9cbbe1bf0b6b0821bc47635a33e, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [70477] core::grantRole(0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, rlcm: [0xa0Cb889707d426A7A386870A03bc70d1b0697598])
    │   ├─ emit RoleGranted(role: 0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, account: rlcm: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0x894017c09ca64151bdcdea52a1f0165fafbd24d1a299ee68541fc9b08b334dd1, term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb])
    │   ├─ emit RoleGranted(role: 0x894017c09ca64151bdcdea52a1f0165fafbd24d1a299ee68541fc9b08b334dd1, account: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [92377] core::grantRole(0x2434c62dad7115c02a27c0967b7aef60b4a026459092461dcb99efc53094cc7e, term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb])
    │   ├─ emit RoleGranted(role: 0x2434c62dad7115c02a27c0967b7aef60b4a026459092461dcb99efc53094cc7e, account: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [4104] core::renounceRole(0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleRevoked(role: 0x7935bd0ae54bc31f548c14dba4d37c5c64b3f8ca900cb468fb8abd54d5894f55, account: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   └─ ← ()
    ├─ [25045] guild::setMaxGauges(10)
    │   ├─ [643] core::hasRole(0x08e5a6ba998e059c55ada17d321c304b7ab7f9cbbe1bf0b6b0821bc47635a33e, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   └─ ← true
    │   ├─ emit MaxGaugesUpdate(oldMaxGauges: 0, newMaxGauges: 10)
    │   └─ ← ()
    ├─ [96914] guild::addGauge(1, term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb])
    │   ├─ [643] core::hasRole(0xf213c52f17fbe7c12e60dedf809e77244577de25e041307d406900a545756831, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   └─ ← true
    │   ├─ emit AddGauge(gauge: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], gaugeType: 1)
    │   └─ ← 0
    ├─ [50159] guild::mint(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 40000000000000000000000000 [4e25])
    │   ├─ [643] core::hasRole(0xe1c4c7c8669e29e308967879c7054b1e9477115fb5fee6072d66b6c291b29545, test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) [staticcall]
    │   │   └─ ← true
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 40000000000000000000000000 [4e25])
    │   └─ ← ()
    ├─ [207565] guild::incrementGauge(term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], 20000000000000000000000000 [2e25])
    │   ├─ [1935] profitManager::claimGaugeRewards(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb])
    │   │   ├─ [837] guild::getUserGaugeWeight(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb]) [staticcall]
    │   │   │   └─ ← 0
    │   │   └─ ← 0
    │   ├─ emit IncrementGaugeWeight(user: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], gauge: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], weight: 20000000000000000000000000 [2e25])
    │   └─ ← 20000000000000000000000000 [2e25]
    ├─ [0] VM::label(core: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], "core")
    │   └─ ← ()
    ├─ [0] VM::label(profitManager: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], "profitManager")
    │   └─ ← ()
    ├─ [0] VM::label(collateral: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], "collateral")
    │   └─ ← ()
    ├─ [0] VM::label(credit: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], "credit")
    │   └─ ← ()
    ├─ [0] VM::label(guild: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], "guild")
    │   └─ ← ()
    ├─ [0] VM::label(rlcm: [0xa0Cb889707d426A7A386870A03bc70d1b0697598], "rlcm")
    │   └─ ← ()
    ├─ [0] VM::label(auctionHouse: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], "auctionHouse")
    │   └─ ← ()
    ├─ [0] VM::label(term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], "term")
    │   └─ ← ()
    ├─ [0] VM::label(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], "test")
    │   └─ ← ()
    └─ ← ()

  [365668] test::testCallMaxDelayBetweenPartialRepay0()
    ├─ [46782] collateral::mint(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 15000000000000000000 [1.5e19])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 15000000000000000000 [1.5e19])
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    ├─ [24651] collateral::approve(term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], 15000000000000000000 [1.5e19])
    │   ├─ emit Approval(owner: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], value: 15000000000000000000 [1.5e19])
    │   └─ ← true
    ├─ [271089] term::borrow(20000000000000000000000 [2e22], 15000000000000000000 [1.5e19])
    │   ├─ [268411] LendingTerm::borrow(20000000000000000000000 [2e22], 15000000000000000000 [1.5e19]) [delegatecall]
    │   │   ├─ [2394] profitManager::creditMultiplier() [staticcall]
    │   │   │   └─ ← 1000000000000000000 [1e18]
    │   │   ├─ [2608] profitManager::minBorrow() [staticcall]
    │   │   │   └─ ← 100000000000000000000 [1e20]
    │   │   ├─ [18221] profitManager::totalBorrowedCredit() [staticcall]
    │   │   │   ├─ [3361] SimplePSM::redeemableCredit() [staticcall]
    │   │   │   │   ├─ [394] profitManager::creditMultiplier() [staticcall]
    │   │   │   │   │   └─ ← 1000000000000000000 [1e18]
    │   │   │   │   └─ ← 0
    │   │   │   ├─ [4576] credit::targetTotalSupply() [staticcall]
    │   │   │   │   └─ ← 0
    │   │   │   └─ ← 0
    │   │   ├─ [2352] profitManager::gaugeWeightTolerance() [staticcall]
    │   │   │   └─ ← 1200000000000000000 [1.2e18]
    │   │   ├─ [9529] guild::calculateGaugeAllocation(term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], 20000000000000000000000 [2e22]) [staticcall]
    │   │   │   └─ ← 20000000000000000000000 [2e22]
    │   │   ├─ [70514] rlcm::mint(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 20000000000000000000000 [2e22])
    │   │   │   ├─ [2643] core::hasRole(0x894017c09ca64151bdcdea52a1f0165fafbd24d1a299ee68541fc9b08b334dd1, term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb]) [staticcall]
    │   │   │   │   └─ ← true
    │   │   │   ├─ emit BufferUsed(amountUsed: 20000000000000000000000 [2e22], bufferRemaining: 340282366920938443463374607431768211455 [3.402e38])
    │   │   │   ├─ [52302] credit::mint(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 20000000000000000000000 [2e22])
    │   │   │   │   ├─ [2643] core::hasRole(0x98eee63452386eeb3e8c10d0cfe42a80ba83b7826c38149de72788766ce2cb36, rlcm: [0xa0Cb889707d426A7A386870A03bc70d1b0697598]) [staticcall]
    │   │   │   │   │   └─ ← true
    │   │   │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 20000000000000000000000 [2e22])
    │   │   │   │   └─ ← ()
    │   │   │   └─ ← ()
    │   │   ├─ [22232] collateral::transferFrom(test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], 15000000000000000000 [1.5e19])
    │   │   │   ├─ emit Approval(owner: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], spender: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], value: 0)
    │   │   │   ├─ emit Transfer(from: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], to: term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb], value: 15000000000000000000 [1.5e19])
    │   │   │   └─ ← true
    │   │   ├─ emit LoanOpen(when: 1679067867 [1.679e9], loanId: 0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a, borrower: test: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], collateralAmount: 15000000000000000000 [1.5e19], borrowAmount: 20000000000000000000000 [2e22])
    │   │   └─ ← 0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a
    │   └─ ← 0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a
    ├─ [665] term::partialRepayDelayPassed(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a) [staticcall]
    │   ├─ [493] LendingTerm::partialRepayDelayPassed(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a) [delegatecall]
    │   │   └─ ← false
    │   └─ ← false
    ├─ [396] term::YEAR() [staticcall]
    │   ├─ [230] LendingTerm::YEAR() [delegatecall]
    │   │   └─ ← 31557600 [3.155e7]
    │   └─ ← 31557600 [3.155e7]
    ├─ [0] VM::warp(1710625467 [1.71e9])
    │   └─ ← ()
    ├─ [0] VM::roll(16848498 [1.684e7])
    │   └─ ← ()
    ├─ [665] term::partialRepayDelayPassed(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a) [staticcall]
    │   ├─ [493] LendingTerm::partialRepayDelayPassed(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a) [delegatecall]
    │   │   └─ ← false
    │   └─ ← false
    ├─ [0] VM::expectRevert(LendingTerm: cannot call)
    │   └─ ← ()
    ├─ [4637] term::call(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a)
    │   ├─ [4451] LendingTerm::call(0xef76992452dcc745eb4787bfb358222c5c1152ec5ff5ecf953055b33567fde0a) [delegatecall]
    │   │   ├─ [702] guild::isDeprecatedGauge(term: [0x03A6a84cD762D9707A21605b548aaaB891562aAb]) [staticcall]
    │   │   │   └─ ← false
    │   │   └─ ← revert: LendingTerm: cannot call
    │   └─ ← revert: LendingTerm: cannot call
    └─ ← ()

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.01ms
```

## Tools Used

Manual review

## Recommended Mitigation Steps

1. Consider adding mechanism to call a loan immediately if loss happened in term
2. Consider setting an upper and lower limit on the time for partial repayment, so that the health of the term is maintained and borrowers continue to make payments
