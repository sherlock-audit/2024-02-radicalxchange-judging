Great Wooden Tadpole

medium

# Some highest bidder's ether will remain on the EnglishPeriodicAuctionFacet smart contract

## Summary

## Vulnerability Detail
After calling the `closeAuction` function, the winning bidder receives the won NFT. If the winner is different from the previous owner, then transfer the highest bid amount to the previous bidder's collateral. 
```solidity
-->   l.availableCollateral[oldBidder] += l
            .highestBids[tokenId][currentAuctionRound].bidAmount;
      l.highestBids[tokenId][currentAuctionRound].collateralAmount = 0;
      l.bids[tokenId][currentAuctionRound][
                l.highestBids[tokenId][currentAuctionRound].bidder
            ].collateralAmount = 0;
```
 Also, distribute the fee to the beneficiary from the highest bidder's fee amount.
 ```solidity
 if (l.highestBids[tokenId][currentAuctionRound].feeAmount > 0) {
            IBeneficiary(address(this)).distribute{
                value: l.highestBids[tokenId][currentAuctionRound].feeAmount
            }();
        }
 ```
 
But the case where a user competed for an NFT with other users and placed multiple bids is not considered, so some ether will remain on the smart contract.
Let's consider the situation where the winner is a user who is different from the previous owner.
1) user `placeBid(tokenId, 1,1e18, {value: 1,1e18 + feeAmount})`;
```solidity
 bid.bidAmount = 1,1e18;
 bid.feeAmount = feeAmount;
bid.collateralAmount = 1,1e18 + feeAmount;

 l.highestBids[tokenId][currentAuctionRound] = bid;
```
2) there were five more bids from other users.
3) user `placeBid(tokenId, 1,7e18, {value: 1,7e18 + feeAmount})`;
```solidity
 bid.bidAmount = 1,7e18;
 bid.feeAmount = feeAmount;
bid.collateralAmount = previousCollateralValue + 1,7e18 + feeAmount;

 l.highestBids[tokenId][currentAuctionRound] = bid;
```

After this transfer the highest bid amount to the previous bidder's collateral - `bid.bidAmount = 1,7e18;`
And distribute fee amount to beneficiary - `bid.feeAmount`

The first bid amount (ether) from the user remained in the smart contract.

## Impact
Some highest bidder's ether will remain on the EnglishPeriodicAuctionFacet smart contract

## Code Snippet
[contracts/auction/EnglishPeriodicAuctionInternal.sol#L343-L348](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L343-L348)

## Tool used

Manual Review

## Recommendation
Consider changing the logic for distributing the remaining ethers.
