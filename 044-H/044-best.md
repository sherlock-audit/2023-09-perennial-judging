Keen Plastic Oyster

high

# MultiInvoker liquidation action will revert most of the time due to incorrect closable amount initialization
## Summary

The fix to [issue 49 of the main contest](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/49) introduced new invalidation system and additional condition: liquidations must close maximum `closable` amount, which is the amount which can be maximally closed based on the latest settled position.

The problem is that `MultiInvoker` incorrectly calculates `closableAmount` (it's not initialized and thus will often return 0 instead of correct magnitude) and thus most `LIQUIDATION` actions will revert.

## Vulnerability Detail

`MultiInvoker` calculates the `closable` amount in its `_latest` function incorrectly. In particular, it doesn't initialize `closableAmount`, so it's set to 0 initially. It then scans pending positions, settling those which should be settled, and reducing `closableAmount` if necessary for remaining pending positions:
```solidity
function _latest(
    IMarket market,
    address account
) internal view returns (Position memory latestPosition, Fixed6 latestPrice, UFixed6 closableAmount) {
    // load parameters from the market
    IPayoffProvider payoff = market.payoff();

    // load latest settled position and price
    uint256 latestTimestamp = market.oracle().latest().timestamp;
    latestPosition = market.positions(account);
    latestPrice = market.global().latestPrice;
    UFixed6 previousMagnitude = latestPosition.magnitude();

    // @audit-issue Should add:
    // closableAmount = previousMagnitude;
    // otherwise if no position is settled in the following loop, closableAmount incorrectly remains 0

    // scan pending position for any ready-to-be-settled positions
    Local memory local = market.locals(account);
    for (uint256 id = local.latestId + 1; id <= local.currentId; id++) {

        // load pending position
        Position memory pendingPosition = market.pendingPositions(account, id);
        pendingPosition.adjust(latestPosition);

        // load oracle version for that position
        OracleVersion memory oracleVersion = market.oracle().at(pendingPosition.timestamp);
        if (address(payoff) != address(0)) oracleVersion.price = payoff.payoff(oracleVersion.price);

        // virtual settlement
        if (pendingPosition.timestamp <= latestTimestamp) {
            if (!oracleVersion.valid) latestPosition.invalidate(pendingPosition);
            latestPosition.update(pendingPosition);
            if (oracleVersion.valid) latestPrice = oracleVersion.price;

            previousMagnitude = latestPosition.magnitude();
@@@         closableAmount = previousMagnitude;

        // process pending positions
        } else {
            closableAmount = closableAmount
                .sub(previousMagnitude.sub(pendingPosition.magnitude().min(previousMagnitude)));
            previousMagnitude = latestPosition.magnitude();
        }
    }
}
```

Notice, that `closableAmount` is initialized to `previousMagnitude` **only if there is at least one position that needs to be settled**. However, if `local.latestId == local.currentId` (which is the case for most of the liquidations - position becomes liquidatable due to price changes without any pending positions created by the user), this loop is skipped entirely, never setting `closableAmount`, so it's incorrectly returned as 0, although it's not 0 (it should be the latest settled position magnitude).

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

All `MultiInvoker` liquidation actions will revert if trying to liquidate users without positions which can be settled, which can happen in 2 cases:
1. Liquidated user doesn't have any pending positions at all (`local.latestId == local.currentId`). This is the most common case (price has changed and user is liquidated without doing any actions) and we can reasonably expect that this will be the case for at least 50% of liquidations (probably more, like 80-90%).
2. Liquidated user does have pending positions, but no pending position is ready to be settled yet. For example, if liquidator commits unrequested oracle version which liquidates user, even if the user already has pending position (but which is not yet ready to be settled).

Since this breaks important `MultiInvoker` functionality in most cases and causes loss of funds to liquidator (revert instead of getting liquidation fee), I believe this should be High severity.

## Code Snippet

There is no initialization of `closableAmount` in `MultiInvoker._latest` before the pending positions loop:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L361-L375

Initialization only happens when settling position:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L393

However, the loop will often be skipped entirely if there are no pending positions at all, thus `closableAmount` will be returned uninitialized (0):
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L376

## Tool used

Manual Review

## Recommendation

Initialize `closableAmount` to `previousMagnitude`:
```solidity
    function _latest(
        IMarket market,
        address account
    ) internal view returns (Position memory latestPosition, Fixed6 latestPrice, UFixed6 closableAmount) {
        // load parameters from the market
        IPayoffProvider payoff = market.payoff();

        // load latest settled position and price
        uint256 latestTimestamp = market.oracle().latest().timestamp;
        latestPosition = market.positions(account);
        latestPrice = market.global().latestPrice;
        UFixed6 previousMagnitude = latestPosition.magnitude();
+       closableAmount = previousMagnitude;
```
