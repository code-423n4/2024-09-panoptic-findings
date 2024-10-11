---
sponsor: "Panoptic"
slug: "2024-09-panoptic"
date: "2024-10-10"
title: "Panoptic Invitational"
findings: "https://github.com/code-423n4/2024-09-panoptic-findings/issues"
contest: 442
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Panoptic smart contract system written in Solidity. The audit took place between September 26—October 3 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 5 wardens participated:

  1. [pkqs90](https://code4rena.com/@pkqs90)
  2. [0xdice91](https://code4rena.com/@0xdice91)
  3. [MrPotatoMagic](https://code4rena.com/@mrpotatomagic)
  4. [K42](https://code4rena.com/@k42)
  5. [KupiaSec](https://code4rena.com/@kupiasec)

This audit was judged by [Picodes](https://code4rena.com/@Picodes).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded 0 vulnerabilities with a risk rating in the categories of HIGH severity or MEDIUM severity.

Additionally, C4 analysis included 2 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Panoptic audit repository](https://github.com/code-423n4/2024-09-panoptic), and is composed of 23 smart contracts written in the Solidity programming language and includes 5,336 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Low Risk and Non-Critical Issues

For this audit, the judge found merit in 2 reports submitted by wardens detailing low risk and non-critical issues. The report highlighted below by **pkqs90** received the top score from the judge.

*The following warden also submitted a report: [0xdice91](https://github.com/code-423n4/2024-09-panoptic-findings/issues/7).*

## [[01] `CollateralTracker` `maxWithdraw`/`maxRedeem` functions may revert in some cases](https://github.com/code-423n4/2024-09-panoptic-findings/issues/3)

### Lines of code

<https://github.com/code-423n4/2024-09-panoptic/blob/main/contracts/CollateralTracker.sol#L489><br>
<https://github.com/code-423n4/2024-09-panoptic/blob/main/contracts/CollateralTracker.sol#L593>

### Impact

CollateralTracker is a ERC4626 compliant vault. However, in some cases, the maxWithdraw/maxRedeem functions may revert.

### Description

According to the README: `Panoptic’s CollateralTracker supports the full ERC4626 interface`. In the scope of this audit, a new change was introduced when calculating maxWithdraw/maxRedeem that it uses `s_poolAssets - 1` as available asset balance since there was 1 wei of virtual balance in the beginning.

```solidity
    function maxWithdraw(address owner) public view returns (uint256 maxAssets) {
        // We can only use the standard 4626 withdraw function if the user has no open positions
        // For the sake of simplicity assets can only be withdrawn through the redeem function
@>      uint256 available = s_poolAssets - 1;
        uint256 balance = convertToAssets(balanceOf[owner]);
        return s_panopticPool.numberOfPositions(owner) == 0 ? Math.min(available, balance) : 0;
    }

    function maxRedeem(address owner) public view returns (uint256 maxShares) {
@>      uint256 available = convertToShares(s_poolAssets - 1);
        uint256 balance = balanceOf[owner];
        return s_panopticPool.numberOfPositions(owner) == 0 ? Math.min(available, balance) : 0;
    }
```

However, it is possible that `s_poolAssets` is zero. When users are minting short positions (equivalent to minting UniswapV3 LP), tokens are sent to UniswapV3. If the amount of tokens happens to be exactly the remainder asset of a CollateralTracker, the updated `s_poolAssets` would turn to zero.

```solidity
    function takeCommissionAddData(
        address optionOwner,
        int128 longAmount,
        int128 shortAmount,
        int128 swappedAmount
    ) external onlyPanopticPool returns (uint32 utilization) {
        unchecked {
            // current available assets belonging to PLPs (updated after settlement) excluding any premium paid
@>          int256 updatedAssets = int256(uint256(s_poolAssets)) - swappedAmount;

            ...

@>          s_poolAssets = uint256(updatedAssets).toUint128();
        }
    }
```

### Recommended Mitigation Steps

For both functions, if `s_poolAssets == 0`, simply return 0.

**[dyedm1 (Panoptic) confirmed and commented](https://github.com/code-423n4/2024-09-panoptic-findings/issues/3#issuecomment-2392426338):**
 > > "However, it is possible that s\_poolAssets is zero. When users are minting short positions (equivalent to minting UniswapV3 LP), tokens are sent to UniswapV3. If the amount of tokens happens to be exactly the remainder asset of a CollateralTracker, the updated s\_poolAssets would turn to zero."
> 
> Good point. This only happens when you can't withdraw anyway though, so this is more of a quirk/incorrect as to spec Low severity issue I think.

**[Picodes (judge) decreased severity to Low and commented](https://github.com/code-423n4/2024-09-panoptic-findings/issues/3#issuecomment-2393107103):**
 > At first sight I think @dyedm1 is right!

***

## [[02] `PanopticPool::_checkSolvencyAtTicks` incorrect code comments](https://github.com/code-423n4/2024-09-panoptic-findings/issues/5)

The code implementation is as follows: if there are multiple `atTicks[]` passed in as parameter:

- For `expectedSolvent == true`, all ticks must be solvent
- For `expectedSolvent == false`, all ticks must be insolvent

However, the code comment `this will return true if solvent at any of the provided tick, and return false iff the account is insolvent at all ticks.` is incorrect and misleading.

```solidity
    /// @notice Check whether an account is solvent at a given `atTick` with a collateral requirement of `buffer`/10_000 multiplied by the requirement of `positionIdList`.
@>  /// @dev this will return true if solvent at any of the provided tick, and return false iff the account is insolvent at all ticks.
    /// @param account The account to check solvency for
    /// @param positionIdList The list of positions to check solvency for
    /// @param currentTick The current tick of the Uniswap pool (needed for fee calculations)
    /// @param atTicks An array of ticks to check solvency at
    /// @param buffer The buffer to apply to the collateral requirement
    /// @param expectedSolvent Whether the account is expected to be solvent (true) or insolvent (false) at all provided `atTicks`
    function _checkSolvencyAtTicks(
        address account,
        TokenId[] calldata positionIdList,
        int24 currentTick,
        int24[] memory atTicks,
        uint256 buffer,
        bool expectedSolvent
    ) internal view {
        ...
        // must combine solvency checks as uint because bool would automatically return false for (false && _check),
        uint8 solvent;
        for (uint256 i; i < numberOfTicks; ) {
            unchecked {
                solvent += (
                    _isAccountSolvent(
                        account,
                        atTicks[i],
                        positionBalanceArray,
                        shortPremium,
                        longPremium,
                        buffer
                    )
                        ? uint8(1)
                        : uint8(0)
                );

                ++i;
            }
        }
        if (expectedSolvent && solvent != numberOfTicks) revert Errors.AccountInsolvent();
        if (!expectedSolvent && solvent != 0) revert Errors.NotMarginCalled();
    }
```

### Code Snippet

- https://github.com/code-423n4/2024-09-panoptic/blob/main/contracts/PanopticPool.sol#L1161

**[dyedm1 (Panoptic) confirmed and commented](https://github.com/code-423n4/2024-09-panoptic-findings/issues/5#issuecomment-2393977275):**
 > Nice catch, will fix.


***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
