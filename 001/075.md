Brisk Burgundy Puma

medium

# Other bidders cannot place bid for `0` amount if the starting bid is `0`.

## Summary

In the current setup, if the starting bid of an auction is set to `0`, only the previous owner retains the ability to initiate the bidding process with a bid amount of `0`.

## Vulnerability Detail

When the starting bid is set to `0`, only the previous owner of the steward license can submit a bid with a value of `0`. Any other prospective bidder is unable to participate because they are required to submit an amount greater than `0`, triggering a function revert due to the validation checks implemented.

```solidity
function _placeBid(
        uint256 tokenId,
        address bidder,
        uint256 bidAmount,
        uint256 collateralAmount
    ) internal {
        EnglishPeriodicAuctionStorage.Layout
            storage l = EnglishPeriodicAuctionStorage.layout();

        uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

        Bid storage bid = l.bids[tokenId][currentAuctionRound][bidder];

        // Check if higher than starting bid
        require(
            bidAmount >= l.startingBid,
            'EnglishPeriodicAuction: Bid amount must be greater than or equal to starting bid'
        );

        // Check if highest bid
        if (l.highestBids[tokenId][currentAuctionRound].bidAmount > 0) {
            require(
                bidAmount >=
                    l.highestBids[tokenId][currentAuctionRound].bidAmount +
                        l.minBidIncrement,
                'EnglishPeriodicAuction: Bid amount must be greater than highest outstanding bid'
            );
        }

        uint256 totalCollateralAmount = bid.collateralAmount + collateralAmount;

        uint256 feeAmount;
        address currentBidder;
        if (IStewardLicense(address(this)).exists(tokenId)) {
            currentBidder = IStewardLicense(address(this)).ownerOf(tokenId);
        } else {
            currentBidder = l.initialBidder;
        }

        if (bidder == currentBidder) {
            // If current bidder, collateral is entire fee amount
            feeAmount = totalCollateralAmount;
        } else {
            require(
 @1>               totalCollateralAmount > bidAmount,
                'EnglishPeriodicAuction: Collateral must be greater than current bid'
            );
            // If new bidder, collateral is bidAmount + fee
            feeAmount = totalCollateralAmount - bidAmount;
        }

        require(
@2>            _checkBidAmount(bidAmount, feeAmount),
            'EnglishPeriodicAuction: Incorrect bid amount'
        );

        ...
    }
```    

GitHub: [[330](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L330)]

They can place bid for higher amount than `0`, but if the user is the only one who wants to bid for the license they should have get it for `0`. but the current design doesn't allow that. Only old owner will be able to vote with bid value of `0`.

## Impact

other bidders will not be able to place bid with `0` amount.

## Code Snippet

#### PoC:

```javascript
    it('other bidders will not be able to bid for 0 bid amount', async function () {
      // Auction start: Now - 200
      // Auction end: Now + 100
      const instance = await getInstance({
        auctionLengthSeconds: 300,
        initialPeriodStartTime: (await time.latest()) - 200,
        licensePeriod: 1000,
        hasOwner: true
      });

      await instance.connect(owner).setStartingBid(ethers.utils.parseEther('0'));

      // place bid
      const bidAmount1 = ethers.utils.parseEther('0');
      const feeAmount1 = await instance.calculateFeeFromBid(bidAmount1);
      const collateralAmount1 = feeAmount1;

      await expect(instance
        .connect(bidder1)
        .placeBid(0, bidAmount1, { value: collateralAmount1 })).to.be.reverted;

    });
```

#### Output:

```bash
AAMIR@Victus MINGW64 /d/radicle-audit/pco-art (master)
$ npx hardhat test --grep 'other bidders will not be able to bid for 0 bid amount'

  EnglishPeriodicAuction
    cancelBid
      ✔ other bidders will not be able to bid for 0 bid amount (716ms)


  1 passing (2s)
```

## Tool used

- Manual Review
- Foundry

## Recommendation
It is recommended to make changes so that the other bidders can place the bid for `0` amount. One way to do it is to check if there is bidder in the highest bidder mapping. 


