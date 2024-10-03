# QA Report

## Low Risk 
| Id | Title |
|:--:|:-------|
| [L-01] | `PanopticPool::getOracleTicks` does not return correct medianData. |
| [I-01] | `PanopticPool::_checkSolvencyAtTicks` incorrect code comments. |

## [L-01] `PanopticPool::getOracleTicks` does not return correct medianData.

### Bug Description

According to the code comments, the `getOracleTicks()` function should return the updated medianData. However, it still returns the old data stored in `s_miniMedian`.

```solidity
    /// @notice Computes and returns all oracle ticks.
    /// @return currentTick The current tick in the Uniswap pool
    /// @return fastOracleTick The fast oracle tick computed as the median of the past N observations in the Uniswap Pool
    /// @return slowOracleTick The slow oracle tick (either composed of Uniswap observations or tracked by `s_miniMedian`)
    /// @return latestObservation The latest observation from the Uniswap pool
    /// @return medianData The updated value for `s_miniMedian` (0 if MEDIAN_PERIOD not elapsed) if `pokeMedian` is called at the current state
    function getOracleTicks()
        external
        view
        returns (
            int24 currentTick,
            int24 fastOracleTick,
            int24 slowOracleTick,
            int24 latestObservation,
            uint256 medianData
        )
    {
        (currentTick, fastOracleTick, slowOracleTick, latestObservation, ) = PanopticMath
            .getOracleTicks(s_univ3pool, s_miniMedian);
        medianData = s_miniMedian;
    }
```

### Code Snippet

- https://github.com/code-423n4/2024-09-panoptic/blob/main/contracts/PanopticPool.sol#L1353

### Recommendation

```solidity
        (currentTick, fastOracleTick, slowOracleTick, latestObservation, medianData) = PanopticMath
            .getOracleTicks(s_univ3pool, s_miniMedian);
        if (medianData == 0) {
            medianData = s_miniMedian;
        }
```

## [I-01] `PanopticPool::_checkSolvencyAtTicks` incorrect code comments.

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