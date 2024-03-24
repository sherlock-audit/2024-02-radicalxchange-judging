Curly Hemp Alligator

medium

# Potential griefing attack and other negative impacts if minBidIncrement is set to 0.

## Summary

The protocol gives artists full freedom to set any of initialization values as they desire. Which means that `0` can be set as a valid value for `minBidIncrement`.

When this happens, a user bidding the exact same amount as the highest bidder will take his place even though he did not bid more. This is because the code implicitly assumes that `minBidIncrement` will always be greater than `0`.

This has 3 negative effects:
- A malicious user can win an auction without bidding more than the highest bidder, thereby breaking a core protocol invariant.
- The previous highest bidder loses the defensive advantage of having the highest bid and is forced to either do the same as the malicious user or increase the price to the point the malicious user cannot match it.  
- A malicious user with multiple accounts can infinitely extend the auction by making identical bids from different accounts right before the deadline.   


## Vulnerability Detail

In `_placeBid` there is a check to ensure the bidder has bid more than the current highest bid. 
 ```solidity 
        if (l.highestBids[tokenId][currentAuctionRound].bidAmount > 0) {
            require(
                bidAmount >=
                    l.highestBids[tokenId][currentAuctionRound].bidAmount +
                        l.minBidIncrement,
                'EnglishPeriodicAuction: Bid amount must be greater than highest outstanding bid'
            );
 ```
As long as `minBidIncrement > 0` the bidAmount will always be greater than the highest outstanding bid or cause a revert. 

But when `minBidIncrement == 0`, a user bidding the exact same amount as the highest bid will not cause a revert since the check is **greater or equal** and will thus be set as the highest bidder. 
  
## Impact

There are 3 impacts:

1. A malicious user can win the auction by bidding the same as the highest bidder, thereby breaking a core protocol invariant. 
2. A normal user who notices the actions of the malicious user, is forced to either do the same and hope the malicious user gives up or he must increase the bid to the point the malicious does not wish to match him. Thereby reversing the normal defensive advantage of the highest bidder. 
3. A malicious user with multiple accounts can abuse this as a griefing attack to extend the auction duration by making identical bids before the deadline. Given that the protocol will be deployed on L2s with very gas costs, this is a realistic attack scenario. 



## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L305-L313

## Tool used

Manual Review

## Recommendation

Change the check to the below in order to account for the case of having a `minBidIncrement == 0`

```solidity
        // Check if highest bid
        if (l.highestBids[tokenId][currentAuctionRound].bidAmount > 0) {
            if(l.minBidIncrement>0){
                require(
                    bidAmount >= l.highestBids[tokenId][currentAuctionRound].bidAmount + l.minBidIncrement,
                    'EnglishPeriodicAuction: Bid amount must be greater than highest outstanding bid'
                );
            } else {
                require(
                bidAmount > l.highestBids[tokenId][currentAuctionRound].bidAmount
                'EnglishPeriodicAuction: Bid amount must be greater than highest outstanding bid'
                );
            }
            ); 
```
