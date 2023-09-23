Keen Plastic Oyster

medium

# Oracle `status()` and `latest()` can return invalid version while updating to a new oracle provider
## Summary

The fix to [issue 46 of the main contest](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/46) created a new issue described below.
When updating to a new oracle provider, the previous provider has to be commited at or after the last requested version to finalize the switch. However, if it's commited unrequested at the timestamp **after** the last requested version, and new provider doesn't have a fresh commit yet, then `oracle.status()` and `oracle.latest()` will return **invalid** version as the latest (`oracle.latest().price = 0`). This breaks important invariant that the oracle latest version must be **valid** (by definition - `latest()` is the last **valid** oracle version).

## Vulnerability Detail

Scenario to make `oracle.latest()` return invalid version:
- `provider1.latest.timestamp = 100`
- `provider2.latest.timestamp = 100`

T=150: `Market.update` is called which calls `oracle.request()`, creating a request for timestamp = 200
T=160: `Oracle.update(provider2)` is called by admin: since last request is not yet commited, `latest()` is still returned from `provider1`.
T=320: `provider1.commit()` with timestamp = 220 is called, commiting an oracle version = 220 (oracle version = 200 is invalid)

At this time, `oracle.latest()` will return invalid oracle version due to these lines in `oracle._handleLatest`:
```solidity
if (!isLatestStale && latestVersion.timestamp > latestOracleTimestamp)
    return at(latestOracleTimestamp);
```

`isLatestStale` is false (because `provider2.latest().timestamp < 200`)
`latestVersion.timestamp = 220` (last commit for `provider1`)
`latestOracleTimestamp = 200` (last request for `provider1`)
So this means `_handleLatest()` (and `latest()`) will return `oracle.at(200)` which is invalid.

## Impact

`oracle.status()` and `oracle.latest()` return invalid oracle version (`price = 0`). This breaks an important invariant that the latest oracle must be the last **valid** oracle version. `Market` handles invalid `oracle.at()` calls correctly (replacing price with `global.latestPrice` if oracleVersion returned is invalid), however it doesn't have any checks for validity of `oracle.status()` assuming it to be always valid. This can cause all kinds of unexpected behavior, such as being able to withdraw (steal) or unfairly liquidate based on `latestVersion.price = 0`.

## Code Snippet

`Oracle._handleLatest` doesn't check if oracle version returned by `at()` is valid or not:
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L117-L118

## Tool used

Manual Review

## Recommendation

This one is rather tricky to fix. The best fix should check if `at()` returned in `_handleLatest` is valid, and if not - return the previous valid version. However, there is no such function (to return the latest valid version prior to a given timestamp). A simple but not fully correct solution is to store (cache) current latest version at `update` time, and then update this cached version if `provider1.latest.timestamp` less than last request timestamp and not update if it's greater than last request timestamp. Then return this cached version if `at()` returns invalid version. But it will return pre-latest version if commit unrequested is made twice (one - before request timestamp, and one - after). However since that's a very rare situation and even if it happens - that'll just be a version incorrect by a few seconds, so probably acceptable.
