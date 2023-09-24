Keen Plastic Oyster

medium

# MultiInvoker liquidation action will revert due to incorrect closable amount calculation for invalid oracle versions
## Summary

The fix to [issue 49 of the main contest](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/49) introduced new invalidation system and additional condition: liquidations must close maximum `closable` amount, which is the amount which can be maximally closed based on the latest settled position.

The problem is that `MultiInvoker` incorrectly calculates `closableAmount` when settling invalid oracle positions and thus `LIQUIDATION` actions will revert in these cases.

## Vulnerability Detail

`MultiInvoker` calculates the `closable` amount in its `_latest` function. This function basically repeats the logic of `Market._settle`, but fails to repeat it correctly for the invalid oracle version settlement. When invalid oracle version is settled, `latestPosition` invalidation should increment, but the `latestPosition` should remain the same. This is achieved in the `Market._processPositionLocal` by adjusting `newPosition` after invalidation before the `latestPosition` is set to `newPosition`:
```solidity
if (!version.valid) context.latestPosition.local.invalidate(newPosition);
newPosition.adjust(context.latestPosition.local);
...
context.latestPosition.local.update(newPosition);
```

However, `MultiInvoker` doesn't adjust the new position and simply sets `latestPosition` to new position both when oracle is valid or invalid:
```solidity
if (!oracleVersion.valid) latestPosition.invalidate(pendingPosition);
latestPosition.update(pendingPosition);
```

This leads to incorrect value of `closableAmount` afterwards:
```solidity
previousMagnitude = latestPosition.magnitude();
closableAmount = previousMagnitude;
```

For example, if `latestPosition.market = 10`, `pendingPosition.market = 0` and pendingPosition has invalid oracle, then:
- `Market` will invalidate (`latestPosition.invalidation.market = 10`), adjust (`pendingPosition.market = 10`), set `latestPosition` to new `pendingPosition` (`latestPosition.maker = pendingPosition.maker = 10`), so `latestPosition.maker` correctly remains 10.
- `MultiInvoker` will invalidate (`latestPosition.invalidation.market = 10`), and immediately set `latestPosition` to `pendingPosition` (`latestPosition.maker = pendingPosition.maker = 0`), so `latestPosition.maker` is set to 0 incorrectly.

Since `LIQUIDATE` action of `MultiInvoker` uses `_latest` to calculate `closableAmount` and `liquidationFee`, these values will be calculated incorrectly and will revert when trying to update the market. See the `_liquidate` market update reducing `currentPosition` by `closable` (which is 0 when it must be bigger):
```solidity
market.update(
    account,
    currentPosition.maker.isZero() ? UFixed6Lib.ZERO : currentPosition.maker.sub(closable),
    currentPosition.long.isZero() ? UFixed6Lib.ZERO : currentPosition.long.sub(closable),
    currentPosition.short.isZero() ? UFixed6Lib.ZERO : currentPosition.short.sub(closable),
    Fixed6Lib.from(-1, liquidationFee),
    true
);
```

This line will revert because `Market._invariant` verifies that `closableAmount` must be 0 after updating liquidated position:
```solidity
if (protected && (
@@@ !closableAmount.isZero() ||
    context.latestPosition.local.maintained(
        context.latestVersion,
        context.riskParameter,
        collateralAfterFees.sub(collateral)
    ) ||
    collateral.lt(Fixed6Lib.from(-1, _liquidationFee(context, newOrder)))
)) revert MarketInvalidProtectionError();
```

## Impact

If there is an invalid oracle version during pending position settlement in `MultiInvoker` liquidation action, it will incorrectly revert and will cause loss of funds for the liquidator who should have received liquidation fee, but reverts instead.

Since this breaks important `MultiInvoker` functionality in some rare edge cases (invalid oracle version, user has unsettled position which should settle during user liquidation with `LIQUIDATION` action of `MultiInvoker`), this should be a valid medium finding.

## Code Snippet

Latest position is calculated incorrectly in `MultiInvoker`:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L388-L389

## Tool used

Manual Review

## Recommendation

Both `Market` and `MultiInvoker` handle position settlement for invalid oracle versions incorrectly (`Market` issue with this was reported separately as it's completely different), so both should be fixed and the fix of this one will depend on how the `Market` bug is fixed. The way it is, `MultiInvoker` correctly adjusts pending position before invalidating `latestPosition` (which `Market` fails to do), however after such action `pendingPosition` must not be adjusted, because it was already adjusted and new adjustment should only change it by the difference from the last invalidation. The easier solution would be just not to change `latestPosition` in case of invalid oracle version, so the fix might be like this (just add `else`):
```solidity
    if (!oracleVersion.valid) latestPosition.invalidate(pendingPosition);
    else latestPosition.update(pendingPosition);
```

However, if the `Market` bug is fixed the way I proposed it (by changing `invalidate` function to take into account difference in invalidation of `latestPosition` and `pendingPosition`), then this fix will still be incorrect, because invalidate will expect unadjusted `pendingPosition`, so in this case `pendingPosition` should not be adjusted after loading it, but it will have to be adjusted for positions not yet settled. So the fix might look like this:
```solidity
    Position memory pendingPosition = market.pendingPositions(account, id);
-   pendingPosition.adjust(latestPosition);

    // load oracle version for that position
    OracleVersion memory oracleVersion = market.oracle().at(pendingPosition.timestamp);
    if (address(payoff) != address(0)) oracleVersion.price = payoff.payoff(oracleVersion.price);

    // virtual settlement
    if (pendingPosition.timestamp <= latestTimestamp) {
        if (!oracleVersion.valid) latestPosition.invalidate(pendingPosition);
-        latestPosition.update(pendingPosition);
+       else {
+           pendingPosition.adjust(latestPosition);
+           latestPosition.update(pendingPosition);
+       }
        if (oracleVersion.valid) latestPrice = oracleVersion.price;

        previousMagnitude = latestPosition.magnitude();
        closableAmount = previousMagnitude;

    // process pending positions
    } else {
+       pendingPosition.adjust(latestPosition);
        closableAmount = closableAmount
            .sub(previousMagnitude.sub(pendingPosition.magnitude().min(previousMagnitude)));
        previousMagnitude = latestPosition.magnitude();
    }
```