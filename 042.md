Boxy Burlap Opossum

medium

# Fee setter can brick placing bids with 0 fee denominator

## Summary

Artist/creators can set the fee denominator to 0. This will cause any call to EnglishPeriodicAuctionFacet.placeBid() to revert because the fee denominator is used as a denominator in a division formula leading to placeBid() causing a revert.

It's important to note that although an artist/creator can set a very high fee for the auction, a whale can still place a bid for the token. By setting the fee denominator to zero, any call to place a bid will revert. This prevents ALL users from making a bid. I believe this goes beyond the scope of the artist/creator's role of setting fees.

## Vulnerability Detail

The fee denominator is set by the artist/creator. Although the setting the fee is out-of-scope, I will mention that there are no checks to see if the denominator is zero.

```solidity
function _setFeeDenominator(uint256 feeDenominator) internal {
    PeriodicPCOParamsStorage.Layout storage l = PeriodicPCOParamsStorage
        .layout();
    l.feeDenominator = feeDenominator;

    emit FeeDenominatorSet(feeDenominator);
}
```

With that in mind, a creator can maliciously set this value to 0. When the denominator is set to 0 and EnglishPeriodicAuctionFacet.placeBid() is called, the following internal function will be called:

```solidity
function _calculateFeeFromBid(
    uint256 bidAmount
) internal view returns (uint256) {
    uint256 feeNumerator = IPeriodicPCOParamsReadable(address(this))
        .feeNumerator();
    uint256 feeDenominator = IPeriodicPCOParamsReadable(address(this))
        .feeDenominator();

    // AUDIT: no checks that the denominator is zero. This can result in a divide by zero revert.
    return (bidAmount * feeNumerator) / feeDenominator;
}
```

As you can see, there are no checks that the denominator is zero. Because of this, a divide by zero revert can occur.

Below is a forge test which shows this bug in action:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/auction/facets/EnglishPeriodicAuctionFacet.sol";
import "../contracts/auction/IEnglishPeriodicAuctionInternal.sol";
import "../contracts/allowlist/facets/AllowListFacet.sol";
import "../contracts/license/IStewardLicense.sol";
import "../contracts/allowlist/IAllowlistReadable.sol";
import "../contracts/pco/IPeriodicPCOParamsReadable.sol";
import "../contracts/beneficiary/IBeneficiary.sol"; 

contract EnglishPeriodicAuctionTest is Test {
  EnglishPeriodicAuctionFacet auction;

  address bidder = address(0x07);
  uint tokenId = 1;
  uint licensePeriod = 1000;

  function setUp() public {
    auction = new EnglishPeriodicAuctionFacet();
    auction.initializeAuction(
      address(0), // repossessor
      address(0), // initialBidder
      0, // initialPeriodStartTime
      0, // initialPeriodStartTimeOffset
      1 ether, // startingBid
      1 days, // auctionLengthSeconds
      0.01 ether, // minBidIncrement
      15 minutes, // bidExtensionWindowLengthSeconds
      5 minutes // bidExtensionSeconds
    );

    
    // allow bidder to make calls
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IAllowlistReadable.isAllowed.selector), 
      abi.encode(true)
    );

    // IStewardLicense(address(this)).exists(tokenId)
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IStewardLicense.exists.selector), 
      abi.encode(tokenId)
    );

    // IPeriodicPCOParamsReadable(address(this)).licensePeriod();
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IPeriodicPCOParamsReadable.licensePeriod.selector),
      abi.encode(licensePeriod)
    );

    // IStewardLicense(address(this)).triggerTransfer
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IStewardLicense.triggerTransfer.selector),
      abi.encode()
    );

    // IStewardLicense(address(this)).ownerOf(tokenId);
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IERC721.ownerOf.selector),
      abi.encode(address(0))
    );

    // IBeneficiary(address(this)).distribute
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IBeneficiary.distribute.selector),
      abi.encode()
    );

    // IPeriodicPCOParamsReadable(address(this)).feeNumerator();
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IPeriodicPCOParamsReadable.feeNumerator.selector),
      abi.encode(1)
    );

    // IPeriodicPCOParamsReadable(address(this)).feeDenominator();
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IPeriodicPCOParamsReadable.feeDenominator.selector),
      abi.encode(10)
    );

    // IStewardLicense(address(this)).maxTokenCount()
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IStewardLicense.maxTokenCount.selector),
      abi.encode(1e18)
    );
  }

  function testZeroDenominatorInFees() public {
    // IPeriodicPCOParamsReadable(address(this)).feeNumerator();
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IPeriodicPCOParamsReadable.feeNumerator.selector),
      abi.encode(1)
    );

    // IPeriodicPCOParamsReadable(address(this)).feeDenominator();
    vm.mockCall(
      address(auction), 
      abi.encodeWithSelector(IPeriodicPCOParamsReadable.feeDenominator.selector),
      abi.encode(0) // AUDIT: denominator set to zero
    );

    deal(bidder, 100e18);

    vm.prank(bidder);
    uint bidAmount = 100e18 - 1;
    uint feeAmount = 9999999999999999999;
    vm.expectRevert();
    auction.placeBid{value: bidAmount}(tokenId, bidAmount + feeAmount);
  }
}
```

## Impact

Artist/creator can maliciously brick the auction system for a given token leading to no bidder being able to bid on the token.  

## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol?plain=1?plain=1#L594-L603

## Tool used

Manual Review

## Recommendation

The team should add a check in EnglishPeriodicAuctionInternal._calculateFeeFromBid() to see if the denominator is zero. If the denominator is zero, then assume that fees should also be zero. 
