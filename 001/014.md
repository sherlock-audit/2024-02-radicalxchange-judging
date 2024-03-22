Curly Hemp Alligator

high

# Highest bidder can withdraw his collateral due to a missing check in _cancelAllBids

## Summary

A bidder with the highest bid cannot cancel his bid since this would break the auction. A check to ensure this was implemented in `_cancelBid`.

However, this check was not implemented in `_cancelAllBids`, allowing the highest bidder to withdraw his collateral and win the auction for free.  

## Vulnerability Detail

The highest bidder should not be able to cancel his bid, since this would break the entire auction mechanism. 

In `_cancelBid` we can find a require check that ensures this:

```solidity
        require(
            bidder != l.highestBids[tokenId][round].bidder,
            'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
        );

```
Yet in `_cancelAllBids`, this check was not implemented. 
```solidity
     * @notice Cancel bids for all rounds
     */
    function _cancelAllBids(uint256 tokenId, address bidder) internal {
        EnglishPeriodicAuctionStorage.Layout
            storage l = EnglishPeriodicAuctionStorage.layout();

        uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

        for (uint256 i = 0; i <= currentAuctionRound; i++) {
            Bid storage bid = l.bids[tokenId][i][bidder];

            if (bid.collateralAmount > 0) {
                // Make collateral available to withdraw
                l.availableCollateral[bidder] += bid.collateralAmount;

                // Reset collateral and bid
                bid.collateralAmount = 0;
                bid.bidAmount = 0;
            }
        }
    }

```
Example: 
User Bob bids 10 eth and takes the highest bidder spot. 
Bob calls `cancelAllBidsAndWithdrawCollateral`.

The `_cancelAllBids` function is called and this makes all the collateral from all his bids from every round available to Bob. This includes the current round `<=` and does not check if Bob is the current highest bidder. Nor is `l.highestBids[tokenId][round].bidder` reset, so the system still has Bob as the highest bidder. 

Then `_withdrawCollateral` is automatically called and Bob receives his 10 eth  back. 

The auction ends. If Bob is still the highest bidder, he wins the auction and his bidAmount of 10 eth is added to the availableCollateral of the oldBidder. 

If there currently is more than 10 eth in the contract (ongoing auctions, bids that have not withdrawn), then the oldBidder can withdraw 10 eth. But this means that in the future a withdraw will fail due to this missing 10 eth. 

## Impact

A malicious user can win an auction for free. 

Additionally, either the oldBidder or some other user in the future will suffer the loss.  

If this is repeated multiple times, it will drain the contract balance and all users will lose their locked collateral. 

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L436

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L380-L413

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L468-L536

## Tool used

Manual Review

## Recommendation

Implement the require check from _cancelBid to _cancelAllBids.
