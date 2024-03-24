Mini Mossy Iguana

medium

# The license period needs to be waited to start a new round even if the previous round had no bidders

## Summary
The license period needs to be waited to start a new round even if the previous round had no bidders.

## Vulnerability Detail
When an auction round is closed the protocol always sets a `licensePeriod` after which another auction round will start. As we can see [here](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L482-L483) this also happens when a round is closed with no bidders, meaning the `repossessor` receives the NFT in license and `licensePeriod` needs to be waited before a new auction round can start.

Since there are no bidders, the NFT should not be given in license.

## Impact
The reposessor will have an NFT of an auction without bidders in license and `licensePeriod` has to be waited before a new round can start, which can economically impact the artist.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Implement a different system to start a new round when the previous round ended with no bidders. This can be a manual system, a round starting immediately again or a custom waiting period that is different from the license period.
