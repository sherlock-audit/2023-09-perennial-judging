Keen Plastic Oyster

medium

# During oracle provider switch, if previous provider feed stops working completely, oracle and market will be stuck with user funds locked in the contract
## Summary

The [issue 46 of the main contest](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/46) after the fix still stands with a more severe condition as described by WatchPug in fix review:
> If we assume it's possible for the previous Python feed to experience a more severe issue: instead of not having an eligible price for the requested oracleVersion, the feed completely stopped working after the requested time, making it impossible to find ANY valid price at a later time than the last requested time, this issue would still exist.

Sponsor response still implies that the previous provider feed **is available**, as they say non-requested version could be posted, but if this feed is no longer available, it will be impossible to commit unrequested, because there will be no pyth price and signature to commit.

> if the previous oracle’s underlying off-chain feed goes down permanently, once the grace period has passed, a non-requested version could be posted to the previous oracle, moving its latest() forward to that point, allowing the switchover to complete.

## Vulnerability Detail

When the oracle provider is updated (switched to a new provider), the latest status (price) returned by the oracle will come from the previous provider until the last request is commited for it, only then the price feed from the new provider will be used. However, it can happen that pyth price feed stops working completely before (or just after) the oracle is updated to a new provider. This means that valid price with signature for **any timestamp after the last request** is not available. In this case, the oracle price will be stuck, because it will ignore new provider, but the previous provider can never finalize (commit a fresh price). As such, the oracle price will get stuck and will never update, breaking the whole protocol with user funds stuck in the protocol.

## Impact

Switching oracle provider can make the oracle stuck and stop updating new prices. This will mean the market will become stale and will revert on all requests from user, disallowing to withdraw funds, bricking the contract entirely.

## Code Snippet

`Oracle._latestStale` will always return false due to this line (since `latest().timestamp` can never advance without a price feed):
https://github.com/sherlock-audit/2023-09-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/Oracle.sol#L128

## Tool used

Manual Review

## Recommendation

Consider ignoring line 128 in `Oracle._latestStale` if a certain timeout has passed after the switch (`block.timestamp - oracles[global.latest].timestamp > SWITCH_TIMEOUT`). This will allow the switch to proceed after some timeout even if previous provider remains uncommited.
