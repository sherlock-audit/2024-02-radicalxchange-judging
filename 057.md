Mini Mossy Iguana

medium

# Changing parameters while a round is ongoing can lead to unexpected behaviour

## Summary
The protocol allows to change auction parameters while a round is ongoing, which can lead to unexpected behaviour. 

## Vulnerability Detail
The protocol allows auction parameters to be changed at any point in time. Parameters can be changed via: 
- [setAuctionsParameters()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/facets/EnglishPeriodicAuctionFacet.sol#L97) 
- [setFeeNumerator()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/pco/facets/PeriodicPCOParamsFacet.sol#L108) and [setFeeDenominator()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/pco/facets/PeriodicPCOParamsFacet.sol#L124)
- [setLicensePeriod()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/pco/facets/PeriodicPCOParamsFacet.sol#L92)

There are some issues that might arise when parameters are changed while an auction is ongoing:
- Reducing the fee amount might allow a bidder to get the highest bid by paying less ETH than the previous bidder, this should never be possible unless the bidder is the current owner.
- Changing the auction length seconds might re-open an already ended auction (given it's ended but not closed yet), or can extend an auction that's about to finish.
- Changing the license period length can result in users being able to get the NFT in license for longer or shorter than they expected when bidding

## Impact
Changing auction parameters while a round is ongoing can lead to unexpected behaviour.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Cache the current auction parameters during the first bid of a round and then adjust the protocol to use those for the rest of the round.