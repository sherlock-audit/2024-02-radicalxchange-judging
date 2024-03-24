Micro Snowy Koala

medium

# Auction extension doesn't work as intended

## Summary

Auction rounds can be extended, when:

```solidity
if (

@>>>  auctionEndTime >= block.timestamp &&

            auctionEndTime - block.timestamp <

            _bidExtensionWindowLengthSeconds()

        ) {

            uint256 auctionLengthSeconds;

            if (l.currentAuctionLength[tokenId] == 0) {

                auctionLengthSeconds = _auctionLengthSeconds();

            } else {

                auctionLengthSeconds = l.currentAuctionLength[tokenId];

            }

            // Extend auction
            l.currentAuctionLength[tokenId] =
                auctionLengthSeconds +
                _bidExtensionSeconds();
        }
```

**auctionEndTime >= block.timestamp** - should return true
And **auctionEndTime - block.timestamp <_bidExtensionWindowLengthSeconds()** - should also return true in order to be a valid time for extension.

However when `auctionEndTime == block.timestamp`, an auction can't be extended.


## Vulnerability Detail

Auction extension happens through `placeBid`:

```solidity
function placeBid(uint256 tokenId, uint256 bidAmount) external payable {
        
        require(
            _isAuctionPeriod(tokenId),
            'EnglishPeriodicAuction: can only place bid in auction period'
        );

        require(
            !_isReadyForTransfer(tokenId),
            'EnglishPeriodicAuction: auction is over and awaiting transfer'
        );

        require(
            IAllowlistReadable(address(this)).isAllowed(msg.sender),
            'EnglishPeriodicAuction: sender is not allowed to place bid'
        );

        _placeBid(tokenId, msg.sender, bidAmount, msg.value);
    }
```

Keep in mind here that the auction should have started but shouldn't have ended.
Basically `_isAuctionPeriod` should return true and `_isReadyForTransfer` should return false.

```solidity
function _isAuctionPeriod(uint256 tokenId) internal view returns (bool) {
        if (tokenId >= IStewardLicense(address(this)).maxTokenCount()) {
            return false;
        }
        //slither-disable-next-line timestamp
        return block.timestamp >= _auctionStartTime(tokenId);
    }
```

This returns true when for example:
**block.timestamp = 8th Dec 18:30 > _auctionStartTime = 8th Dec 18:20** - this means auction has started

```solidity
function _isReadyForTransfer(uint256 tokenId) internal view returns (bool) {
        if (tokenId >= IStewardLicense(address(this)).maxTokenCount()) {
            return false;
        }
        //slither-disable-next-line timestamp
        return block.timestamp >= _auctionEndTime(tokenId);
    }
```
This here is used to tell when the auction has ended and the nft is ready to be transferred. Let's go into the internal function and focus on the extension logic:

```solidity
    /**
     * @notice Place a bid
     */
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
                totalCollateralAmount > bidAmount,
                'EnglishPeriodicAuction: Collateral must be greater than current bid'
            );

            // If new bidder, collateral is bidAmount + fee
            feeAmount = totalCollateralAmount - bidAmount;
        }

        require(
            _checkBidAmount(bidAmount, feeAmount),
            'EnglishPeriodicAuction: Incorrect bid amount'
        );

        // Save bid
        bid.bidder = bidder;
        bid.bidAmount = bidAmount;
        bid.feeAmount = feeAmount;
        bid.collateralAmount = totalCollateralAmount;

        l.highestBids[tokenId][currentAuctionRound] = bid;

        emit BidPlaced(tokenId, currentAuctionRound, bid.bidder, bid.bidAmount);

        // Check if auction should extend
        uint256 auctionEndTime = _auctionEndTime(tokenId);

        // slither-disable-start timestamp
        if (

  @>>    auctionEndTime >= block.timestamp &&

            auctionEndTime - block.timestamp <

            _bidExtensionWindowLengthSeconds()

        ) {

            uint256 auctionLengthSeconds;

            if (l.currentAuctionLength[tokenId] == 0) {

                auctionLengthSeconds = _auctionLengthSeconds();

            } else {

                auctionLengthSeconds = l.currentAuctionLength[tokenId];

            }

            // Extend auction
            l.currentAuctionLength[tokenId] =
                auctionLengthSeconds +
                _bidExtensionSeconds();
        }
        // slither-disable-end timestamp
    }
```

Here it is intended to be able to call `_placeBid` even when `auctionEndTime >= block.timestamp`.

Imagine the following scenario:

1. auctionEndTime == block.timestamp(Dec 8th 18:30 == Dec 8th 18:30)
2. When we try to call `placeBid` when both are equal, `_isAuctionPeriod` will return true since the auction has started, but `_isReadyForTransfer` will return true too because in `_isReadyForTransfer`'s check when `return block.timestamp >= _auctionEndTime(tokenId);` means that the auction has concluded therefore when `auctionEndTime == block.timestamp` the call to `placeBid` will revert which should not be the case.

## Impact

Auction extension doesn't work as intended

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L235-L241

## Tool used

Manual Review

## Recommendation
Make the check in `_isReadyForTransfer` to be `return block.timestamp > _auctionEndTime(tokenId);`
