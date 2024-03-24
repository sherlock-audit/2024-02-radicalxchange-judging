Curly Hemp Alligator

high

# Auction winnings are stuck in the contract when oldBidder == address(0) while the license has not yet been minted

## Summary

The `l.initialBidder` variable can be set to `address(0)` when there is no previous bidder. This is a valid input confirmed by the protocol team in the public discord. 

In `_closeAuction`, if the license has not yet been minted (which is a valid path confirmed by documentation and code), then `oldBidder = l.initialBidder`.

This case will happen very often when a artist sets up an auction for a new NFT collection. He will set initialBidder to `address(0)` since there is no previous bidder and he will set the minting of the license conditional on the successful closure of the auction (minting happens through `triggerTransfer`) to avoid gas costs.

The highest bid will be added to `l.availableCollateral[oldBidder]`, which in this case is `address(0)`.

As a consequence, the artist receives nothing and the collateral from the highest bid is locked in the contract. 

## Vulnerability Detail

The `l.initialBidder` can be set during initialization to the address of an initial bidder, or `address(0)` if there is no true initial bidder. The protocol team confirmed in the **public Discord** that this is a valid and sometimes expected value.
![image](https://github.com/sherlock-audit/2024-02-radicalxchange-14si2o/assets/111807030/9733ebbe-41f1-4463-ba1e-366f03ca4e14)

So whenever an artist wishes to put a new collection up for auction, it is very often the case that  `l.initialBidder = address(0)`;


  
When we look at the `_closeAuction` function, we find the below `if` statement

```solidity
       address oldBidder;
        if (IStewardLicense(address(this)).exists(tokenId)) {
            oldBidder = IStewardLicense(address(this)).ownerOf(tokenId);
        } else {
            oldBidder = l.initialBidder;
        }
```
If this resolves to `false`, then `oldBidder` will be set to `address(0)` in the aforementioned scenario.

When does this happen? The protocol allows artists to delay the minting of the license until the end of the auction. This is documented [here](https://pco-art-docs.vercel.app/for-artists/instantiating-your-art/). If an artist decides to do this, then the minting will only happen when the `triggerTransfer` function is called at the very end of `_closeAction`.

In `StewardLicenseInternal: _triggerTransfer`
```solidity
    function _triggerTransfer(
        address from,
        address to,
        uint256 tokenId
    ) internal {
        if (!_exists(tokenId)) {
            // Mint token
            _mint(to, tokenId);
        } else {
            // Safe transfer is not needed. If receiver does not implement ERC721Receiver, next auction can still happen. This prevents a failed transfer from locking up license
            _transfer(from, to, tokenId);
        }
    }

```
So when a artist delays minting and sets no initial bidder, then `oldBidder = address(0)`

This leads to a critical issue in the below line in `_closeAuction`

```solidity
            // Transfer bid to previous bidder's collateral
            l.availableCollateral[oldBidder] += l.highestBids[tokenId][currentAuctionRound].bidAmount;
            l.highestBids[tokenId][currentAuctionRound].collateralAmount = 0;
            l.bids[tokenId][currentAuctionRound][l.highestBids[tokenId][currentAuctionRound].bidder].collateralAmount = 0;
```
The `bidAmount` of the highest bidder is added to the `l.availableCollateral[oldBidder]`. But since oldBidder is address(0) in this scenario, the bidAmount is lost and the collateral sent by the bidder is permanently stuck in the contract. 

## Impact

The winnings of the auction are stuck in the contract with no way of getting it out. 
The artist who put his art up for auction receives nothing. 


## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L472-L477

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L463-L534

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseInternal.sol#L94-L108

## Tool used

Manual Review

## Recommendation

The `initialBidder` has extremely little use in the protocol. I would suggest removing it and replacing `initialBidder` with `repossessor` in `_closeAuction`. Assuming the repossessor is the artist or at least the owner, the winnings will always be sent to the correct person in above described scenario.  
