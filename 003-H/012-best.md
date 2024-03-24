Curly Hemp Alligator

high

# All bids with 100% collateralisation will revert due to incorrect fee calculation

## Summary

Users who wish to participate in an auction must provide full collateralisation (100%) of their bids, while the fee is a percentage of that bid amount. This is clearly stated in the [documentation](https://pco-art-docs.vercel.app/stewards/bidding-in-auction-pitches). 

Yet, due to a check in `_placeBid`, any bid with a collateralisation rate of 100% will revert. 

The root of the issue is the expectation that the fee should be paid **on top** of the bidAmount, which goes fully against documentation and poses many problems. 

- Since the protocol has the stated requirement of 100% collateralisation, many valid bids will revert with the confusing message that you need overcollateralisation.
- The fee is a percentage of the bid, so there is no logical reason why the fee should be paid on top of the bid. 
- Even if users are willing to overcollateralise, for every bid they need to take into account both collateral paid and fee paid and recalculate both perfectly to be within rounding error. This is prone to error.
- In a heated auction with many competing bidders trying to take the price close to a deadline, the complexity demanded to perfectly calculate every bid is not realistic. This will lead to many valid bids being reverted due to a wei-sized deviation and the auction will close at a lower price than the highest valid bid submitted. This constitutes a loss to the creator's circle and the current holder of the license.   



## Vulnerability Detail

In the [documentation](https://pco-art-docs.vercel.app/stewards/bidding-in-auction-pitches), we can see:

```diff
Bidding in a PCO English Auction

A bid submitted in PCOArt's English auction has two parts:

    Bid Value - The price a prospective Steward is willing to pay the current Steward to assume control of the license (or in the case of a current Steward's bid, the payment value they'd be willing to forgo to retain control)
    Honorarium - The monetary contribution passed to a work's Creator Circle based on a percentage of their Bid Value

These bids require full collateralization to ensure proper auction closing and funds distribution.
```

Yet in `_placeBid` we find:   

```solidity
        } else {
            require(
                totalCollateralAmount > bidAmount,
                'EnglishPeriodicAuction: Collateral must be greater than current bid'
            );
            // If new bidder, collateral is bidAmount + fee
            feeAmount = totalCollateralAmount - bidAmount;
 
        }
```
This is the stated scenario from docs
Bob Bid = 100
Collateral = 100
Fee = 5%

Bob wins.  
Creator's circle = 100 * 5% = 5
oldBidder = 100 - 5 = 95   

However, if Bob follows the above scenario, his bid will revert since the above require resolves to `require(100>100)`: `EnglishPeriodicAuction: Collateral must be greater than current bid`


Simply providing some extra collateralisation won't work aswell, since the `_checkBidAmount` further down the function will revert if the added collateral is not within rounding error of the correct fee calculation of `fee % * bidAmount`. 

This becomes impossibly complex when we take a normal auction where Bob & Alice keep bidding against each other in order to take the highest bid. For every new bid, Bob must perfectly! calculate within 1 wei the exact amount of overcollateralisation he needs, which changes every time due to collateral deposited of which part is for the bidAmount and part is for the feeAmount. 

In a time-pressured environment of an auction, this is impossible to do without there being errors. So many auctions will end not because the highest bid won, but because the highest bid made a 1 wei calculation mistake.   


## Impact

- Since the protocol has the stated requirement of 100% collateralisation, many valid bids will revert with the confusing message that you need overcollateralisation.
- The fee is a percentage of the bid, so there is no logical reason why the fee should be paid on top of the bid. 
- Even if users are willing to overcollateralise, for every bid they need to take into account both collateral paid and fee paid and recalculate both perfectly to be within rounding error. This is prone to error.
- In a heated auction with many competing bidders trying to take the price close to a deadline, the complexity demanded to perfectly calculate every bid is not realistic. This will lead to many valid bids being reverted due to a wei-sized deviation and the auction will close at a lower price than the highest valid bid submitted. This constitutes a loss to the creator's circle and the current holder of the license.   


## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L286-L414

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L607-L614
## Tool used

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L587-L599

Manual Review

## Recommendation

Change the check so that `totalCollateralAmount == bidAmount` as anything besides full collateralisation is a mistake. Also fix the fee calculation so that it is a % of bidAmount, not the subtraction between collateral and bid. 