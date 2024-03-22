Mini Mossy Iguana

medium

# The protocol is not compatible with collections of NFTs with non-sequential IDs or sequential IDs that don't start from 0

## Summary
The protocol is not compatible with collections of NFTs with non-sequential IDs or sequential IDs that don't start from 0.

## Vulnerability Detail
The protocol allows to mint an auctionable NFT with a custom ID via [mintToken()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseBase.sol#L31) but the auction system assumes that the IDs of the auctioned NFTs are sequential and starting from 0.

This can cause issues in the following functions:
- [_isAuctionPeriod()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L224-L230), called to know if an NFT with ID `tokenId` can be auctioned, always returns `false` if the `tokenId` is lower than `maxTokenCount`. This means that NFTs minted with an ID bigger than the amount of NFTs currently available in the collection cannot be auctioned.
- [_isReadyForStranfer()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L568) performs the same check as [_isAuctionPeriod()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L224-L230).
- [_auctionStartTime()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L543) can set the auction start time to [`initialPeriodStartTimeOffset * tokenId`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L568), which means an auction for an NFT with a very high ID can potentially be far in the future. 

## Impact
It might be impossible to auction some NFTs. 

## Code Snippet

## Tool used

Manual Review

## Recommendation
Adjust the protocol in such a way that NFTs with non-sequential IDs or IDs not starting from 0 can be used.