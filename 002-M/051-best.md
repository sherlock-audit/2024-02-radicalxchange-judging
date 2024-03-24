Special Carmine Scallop

high

# Token owner's collateral will not be refunded if no one bids in the next auction round.

## Summary
For each auction round, the highest bid at the end of the auction wins, and the bidder gains the ownership of the token for the next license period. By the next round, if the old owner loses the auction, his `bitAmount` will be refunded as `availableCollateral`. But if no one bids in the new auction round, the `bitAmount` will not be refunded, causing the old owner to lose his collateral.

## Vulnerability Detail
As shown in `_closeAuction`, the `oldBidder` (the current owner of the token) will get his `bitAmount` back if he is not the new highest bidder (L500). However, if no one bids in the new auction round (L485), the token will goes to `repossessor`, but `oldBidder` does not get back his `bitAmount` in this case.
```solidity
// EnglishPeriodicAuctionInternal.sol#_closeAuction()

465:    function _closeAuction(uint256 tokenId) internal {
...
485:@>      if (l.highestBids[tokenId][currentAuctionRound].bidder == address(0)) {
486:            // No bids were placed, transfer to repossessor
487:            Bid storage repossessorBid = l.bids[tokenId][currentAuctionRound][
488:                l.repossessor
489:            ];
490:            repossessorBid.bidAmount = 0;
491:            repossessorBid.feeAmount = 0;
492:            repossessorBid.collateralAmount = 0;
493:            repossessorBid.bidder = l.repossessor;
494:
495:            l.highestBids[tokenId][currentAuctionRound] = repossessorBid;
496:        } else if (
497:            l.highestBids[tokenId][currentAuctionRound].bidder != oldBidder
498:        ) {
499:            // Transfer bid to previous bidder's collateral
500:@>          l.availableCollateral[oldBidder] += l
501:            .highestBids[tokenId][currentAuctionRound].bidAmount;
502:            l.highestBids[tokenId][currentAuctionRound].collateralAmount = 0;
503:            l
504:            .bids[tokenId][currentAuctionRound][
505:                l.highestBids[tokenId][currentAuctionRound].bidder
506:            ].collateralAmount = 0;
507:        } else {
508:            l.highestBids[tokenId][currentAuctionRound].collateralAmount = 0;
509:            l
510:            .bids[tokenId][currentAuctionRound][oldBidder].collateralAmount = 0;
511:        }
...
538:    }
```
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L485-L511

The following test case shows the described senario (Insert the `it` function to `EnglishPeriodicAuction.ts#placeBid`).
```TypeScript
    it.only('collateral of old owner is not returned if no one places bid', async function () {
      const repossessor = owner;

      // Auction start: Now - 200
      // Auction end: Now + 100
      const instance = await getInstance({
        auctionLengthSeconds: 300,
        initialPeriodStartTime: (await time.latest()) - 200,
        licensePeriod: 1000,
        initialBidder: repossessor.address,
        repossessor: repossessor.address,
      });

      const licenseMock = await ethers.getContractAt(
        'NativeStewardLicenseMock',
        instance.address,
      );

      // Round 0: bidder1 places bid, and token goes to bidder1
      const bidAmount1 = ethers.utils.parseEther('1.1');
      const feeAmount1 = await instance.calculateFeeFromBid(bidAmount1);
      const collateralAmount1 = feeAmount1.add(bidAmount1);
      await instance.connect(bidder1).placeBid(0, bidAmount1, { value: collateralAmount1 });

      await time.increase(100);
      await instance.connect(bidder1).closeAuction(0);
      const highestBid1 = await instance['highestBid(uint256,uint256)'](0, 0);
      expect(highestBid1.bidAmount).to.be.equal(bidAmount1);
      expect(highestBid1.feeAmount).to.be.equal(feeAmount1);
      expect(highestBid1.collateralAmount).to.be.equal(0);
      expect(highestBid1.bidder).to.be.equal(bidder1.address);
      expect(await licenseMock.ownerOf(0)).to.be.equal(bidder1.address);

      await time.increase(1500);  // licensePeriod + second roundPeriod

      // Round 1: no one places bid, and token goes to repossessor.
      await instance.connect(bidder1).closeAuction(0);
      const highestBid2 = await instance['highestBid(uint256,uint256)'](0, 1);
      expect(highestBid2.bidAmount).to.be.equal(0);
      expect(highestBid2.feeAmount).to.be.equal(0);
      expect(highestBid2.collateralAmount).to.be.equal(0);
      expect(highestBid2.bidder).to.be.equal(repossessor.address);
      expect(await licenseMock.ownerOf(0)).to.be.equal(repossessor.address);

      // Issue: bidder1's available collateral should equal to `bidAmount1`, but it's 0.
      const collateral = await instance['availableCollateral(address)'](bidder1.address);
      expect(collateral).to.be.equal(0);
    });
```

## Impact
If no one bids in the next auction round, the old owner of the token will lose his collaterals.

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L485-L511

## Tool used

Manual Review

## Recommendation
Do not forge to refund `oldBidder`'s collateral even if no one bids in new auction round.
```solidity
485:@>      if (l.highestBids[tokenId][currentAuctionRound].bidder == address(0)) {
486:            // No bids were placed, transfer to repossessor
487:            Bid storage repossessorBid = l.bids[tokenId][currentAuctionRound][
488:                l.repossessor
489:            ];
490:            repossessorBid.bidAmount = 0;
491:            repossessorBid.feeAmount = 0;
492:            repossessorBid.collateralAmount = 0;
493:            repossessorBid.bidder = l.repossessor;
494:
495:            l.highestBids[tokenId][currentAuctionRound] = repossessorBid;
+               if (currentAuctionRound > 0) {
+                   l.availableCollateral[oldBidder] = l.highestBids[tokenId][currentAuctionRound - 1].bidAmount;
+               }
496:        } else if (
```
