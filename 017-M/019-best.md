Mini Mossy Iguana

medium

# An auction of an NFT with `tokenId` equal to 0 can start earlier than expected

## Summary
An auction of an NFT with `tokenId` equal to 0 can start earlier than expected.

## Vulnerability Detail
An auction can incorrectly start without an offset even if an offset has been set if the auctioned NFT has `tokenId` equal to 0. This happens when:
1. It's the first round of an auction
2. `l.tokenInitialPeriodStartTime[0]` has been set to 0

When these two conditions are met the function [_auctionStartTime](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L543) will calculate the auction start time as:
```solidity
auctionStartTime = initialPeriodStartTime + (tokenId * initialPeriodStartTimeOffset);
``` 

If `tokenId` is `0` the offset gets cancelled, making the auction start immediately instead of the correct time. 

## Impact
In some scenarios an auction can start earlier than expected.

## Code Snippet

## Tool used

Manual Review

## Recommendation
In [_auctionStartTime](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L568) calculate the start of an auction by adding `1` to the tokenID:
```solidity
auctionStartTime = initialPeriodStartTime + (tokenId * initialPeriodStartTimeOffset);
```

If any `tokenId` needs to be supported also add a check to make sure the addition doesn't revert in case `tokenId` is equal to `type(uint256).max`.