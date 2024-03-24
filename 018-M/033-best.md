Mini Mossy Iguana

high

# Currently auctioned NFTs can be transferred to a different address in a specific edge case

## Summary
Currently auctioned NFTs can be transferred to a different address in a specific edge case, leading to theft of funds.

## Vulnerability Detail
The protocol assumes that an NFT cannot change owner while it's being auctioned, this is generally the case but there is an exception, an NFT can change owner via [mintToken()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseBase.sol#L31) while an auction is ongoing when all the following conditions apply:
1. An NFT is added to the collection without being minted (ie. `to` set to `address(0)`).
2. The NFT is added to the collection with the parameter `tokenInitialPeriodStartTime[]` set to a timestamp lower than `l.initialPeriodStartTime` but bigger than `0`(ie. `0 < tokenInitialPeriodStartTime[] < l.initialPeriodStartTime`).
3. The current `block.timestamp` is in-between `tokenInitialPeriodStartTime[]` and `l.initialPeriodStartTime`.

A malicious `initialBidder` can take advantage of this by:
1. Bidding on the new added NFT via [placeBid()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L286).
2. Calling [mintToken()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseBase.sol#L31) to transfer the NFT to a different address he controls.
3. Closing the auction via [closeAuction()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L465)

At point `3.`, because the NFT owner changed, the winning bidder (ie. `initialBidder`) is not the current NFT owner anymore. This will trigger the [following line of code](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L499-L506):
```solidity
l.availableCollateral[oldBidder] += l.highestBids[tokenId][currentAuctionRound].bidAmount;
```

Which increases the `availableCollateral` of the `oldBidder` (ie. the address that owns the NFT after point `2.`) by `bidAmount` of the highest bid. But because at the moment the highest bid was placed `initialBidder` was also the NFT owner, he only needed to transfer the `ETH` fee to the protocol instead of the whole bid amount. 

The `initialBidder` is now able to extract ETH from the protocol via the address used in point `2.` by calling [withdrawCollateral()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L439) while also retaining the NFT license.

## Impact
Malicious initial bidder can potentially steal ETH from the protocol in an edge case. If the `ADD_TOKEN_TO_COLLECTION_ROLE` is also malicious, it's possible to drain the protocol.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Don't allow `tokenInitialPeriodStartTime[]` to be set at a timestamp before`l.initialPeriodStartTime`.