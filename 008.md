Keen Plastic Oyster

medium

# Two invalid oracle updates in a row can permanently brick Market contract
## Summary

The fix to [issue 49 of the main contest](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/49) created a new issue described below.
If oracle version is skipped, `_processPositionGlobal` invalidates the latest position by increasing invalidation accumulator, effectively applying inverse delta to all remaining pending positions. However, when calculating invalidation values, the **non-adjusted** `newPosition` values are used:
```solidity
if (!oracleVersion.valid) context.latestPosition.global.invalidate(newPosition);
newPosition.adjust(context.latestPosition.global);
```

Notice that first the the values of **non-adjusted** `newPosition` are used to increment invalidation accumulator, and only then the new invalidation accumulator is applied to `newPosition`. Example when this is wrong:
- `latestPosition.long = 0`
- `latestPostion.invalidation.long = -10`
- `newPosition.long = 10`

In this example, `newPosition.long = 10`, but when adjusted, it should be `newPosition.long = 10 - 10 = 0`. In this case invalidation shouldn't change, because the real (adjusted) `newPosition.long = 0`, but as it's not adjusted and absolute value is used:
- `invalidate(newPosition)` will set `latestPosition.invalidation.long = -10 + (0 - 10) = -20`
- `newPosition.adjust()` will underflow, because `long` is `UFixed6` (unsigned), and it will try to set `newPosition.long = 10 - 20 = -10`

Such situation can happen if there are 2 consecutive invalid oracle versions.

## Vulnerability Detail

Scenario for bricking the entire `Market` contract due to the bug above:
T=1: Global `long = 0`
T=1: Alice requests to open `long = 10` (`long = 10` is added to pending positions at `t=100`)
T=101: Alice settles position (`long = 10` is added to pending positions at `t=200`)
...(no oracle commits)
T=201: Alice settles position (`long = 10` is added to pending positions at `t=300`)
...(still no oracle commits)
T=320: Oracle version = 300 is commited (making oracle versions 100 and 200 invalid)

From this point on - any call to `Market.update` by any user will revert due to underflow as described above in `Market._settle`, bricking the `Market` contract with all user funds locked in it.

Why it will revert? After T=320, global pending positions will be:
- t=100(invalid): long=10
- t=200(invalid): long=10
- t=300(valid): long=10

As described above, when processing these positions, in `_processPositionGlobal`:
1. t=100 (invalid) position: `latestPosition.long = 0`, `latestPosition.invalidation.long = -10`
2. t=200 (invalid) position: `latestPosition.long = 0`, `latestPosition.invalidation.long = -20`. It reverts when trying to adjust `newPosition` (as it has `long = 10` and `invalidation.long = -20`).

## Impact

If 2 or more invalid oracle versions in a row happen, pending positions at the 2nd invalidation or later will have incorrect values, which can make `Market` contract brick permanently, locking up all user funds in the contract without any ability to retrieve them.

## Proof of concept

The scenario above is demonstrated in the test, add this to test/unit/market/Market.test.ts:
```solidity
it('two invalid oracles', async () => {
    const positionMaker = parse6decimal('2.000')
    const positionLong = parse6decimal('0.000')
    const positionLong2 = parse6decimal('1.000')
    const collateral = parse6decimal('1000')

    const oracleVersion = {
    price: parse6decimal('100'),
    timestamp: TIMESTAMP,
    valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion.timestamp).returns(oracleVersion)
    oracle.status.returns([oracleVersion, oracleVersion.timestamp + 100])
    oracle.request.returns()

    dsu.transferFrom.whenCalledWith(userB.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(userB).update(userB.address, positionMaker, 0, 0, collateral, false)

    dsu.transferFrom.whenCalledWith(user.address, market.address, collateral.mul(1e12)).returns(true)
    await market.connect(user).update(user.address, 0, positionLong, 0, collateral, false)

    const oracleVersion2 = {
    price: parse6decimal('100'),
    timestamp: TIMESTAMP + 100,
    valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion2.timestamp).returns(oracleVersion2)
    oracle.status.returns([oracleVersion2, oracleVersion2.timestamp + 100])
    oracle.request.returns()

    // pending position for version3, t=200 (invalid)
    await market.connect(user).update(user.address, 0, positionLong2, 0, 0, false)

    // version3 - invalid, current t=400 (which will also be invalid)
    oracle.status.returns([oracleVersion2, oracleVersion2.timestamp + 200])
    await market.connect(user).update(user.address, 0, positionLong2, 0, 0, false)

    // invalid oracle version
    const oracleVersion3 = {
    price: 0,
    timestamp: TIMESTAMP + 200,
    valid: false,
    }
    oracle.at.whenCalledWith(oracleVersion3.timestamp).returns(oracleVersion3)
    const oracleVersion4 = {
    price: 0,
    timestamp: TIMESTAMP + 300,
    valid: false,
    }
    oracle.at.whenCalledWith(oracleVersion4.timestamp).returns(oracleVersion4)

    // next oracle version is valid
    const oracleVersion5 = {
    price: parse6decimal('100'),
    timestamp: TIMESTAMP + 400,
    valid: true,
    }
    oracle.at.whenCalledWith(oracleVersion5.timestamp).returns(oracleVersion5)

    // still returns oracleVersion2, because nothing commited for version 4, and version 5 time has passed but not yet commited
    oracle.status.returns([oracleVersion2, oracleVersion5.timestamp + 100])
    oracle.request.returns()

    // this one will be valid (version5)
    await market.connect(user).update(user.address, 0, positionLong2, 0, 0, false)
    //await market.connect(user).update(user.address, 0, positionLong2, 0, 0, false)

    // oracleVersion5 commited
    oracle.status.returns([oracleVersion5, oracleVersion5.timestamp + 100])
    oracle.request.returns()

    // bricked here (always reverts for any user as it can't _settle global)
    await expect(            
        market.connect(userB).update(userB.address, positionMaker, 0, 0, 0, false)
    ).to.be.reverted;

    console.log("Market contract is now bricked, reverting any update");
})
```

## Code Snippet

`_processPositionGlobal` invalidates position without first adjusting it:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L401-L402

`_processPositionLocal` does the same:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L447-L448

## Tool used

Manual Review

## Recommendation

When invalidating, `newPosition` should first be adjusted. To avoid adjusting it twice, the easiest fix is probably to change `invalidation` library function `increment`, subtracting adjusted `newPosition` values instead of direct values:
```solidity
    function increment(Invalidation memory self, Position memory latestPosition, Position memory newPosition) internal pure {
        self.maker = self.maker.add(Fixed6Lib.from(latestPosition.maker).sub(Fixed6Lib.from(newPosition.maker).add(self.maker.sub(newPosition.invalidation.maker))));
        self.long = self.long.add(Fixed6Lib.from(latestPosition.long).sub(Fixed6Lib.from(newPosition.long).add(self.long.sub(newPosition.invalidation.long))));
        self.short = self.short.add(Fixed6Lib.from(latestPosition.short).sub(Fixed6Lib.from(newPosition.short).add(self.short.sub(newPosition.invalidation.short))));
    }
```