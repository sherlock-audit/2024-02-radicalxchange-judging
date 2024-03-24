Strong Lava Millipede

high

# The incumbent steward can avoid paying the periodic honorarium.

## Summary

Incumbent stewards, as the current owner of the token under auction, are subject to a unique auction mechanism where the collateral value of their placed bids contribute only towards paying the periodic honorarium, as this eliminates the admittedly redundant process of paying themselves.

However, this execution path can be manipulated by the steward by transferring the token in the time period between winning the auction and settling it, enabling the incumbent steward to win auctions risk free and even turn an attacker-controlled profit in proportion to the periodic honorarium.

## Vulnerability Detail

Firstly, we need to emphasise that whenever an incumbent steward makes a bid via [`placeBid(uint256,address,uint256,uint256)`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L286C14-L291C6), they must specify a value of [`bidAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L289C17-L289C26) and [`collateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L290C17-L290C33) which will ensure that a precise [`feeAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L317C17-L317C26) is paid.

Let's quickly run through how this works.

For an incumbent steward, their computed [`feeAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L317C17-L317C26) is hardcoded to the [`totalCollateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L315C17-L315C38):

```solidity
if (bidder == currentBidder) {
  // If current bidder, collateral is entire fee amount
  feeAmount = totalCollateralAmount;
}
```

To ensure the transaction doesn't revert with `EnglishPeriodicAuction: Incorrect bid amount`, the incumbent steward call is expected provide an equivalent [`bidAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L289C17-L289C26) for the collateral value that will guarantee the call to [`_checkBidAmount(uint256, uint256)`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L608C14-L611C6) will satisfy the following strict equality check:

```solidity
/**
 * @notice Check that fee is within rounding error of bid amount
 */
function _checkBidAmount(
    uint256 bidAmount,
    uint256 feeAmount
) internal view returns (bool) {
    uint256 calculatedFeeAmount = _calculateFeeFromBid(bidAmount);

    return calculatedFeeAmount == feeAmount;
}
```

Here, the [`feeAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L610C17-L610C26) (which we now know [is hard-wired](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L327C25-L327C46) to the [`totalCollateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L315C17-L315C38)) must equal the result of running [`_calculateFeeFromBid(uint256)`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L594C14-L596C6) on the caller-supplied [`bidAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L289C17-L289C26) value:

```solidity
/**
 * @notice Calculate fee from bid
 */
function _calculateFeeFromBid(
    uint256 bidAmount
) internal view returns (uint256) {
    uint256 feeNumerator = IPeriodicPCOParamsReadable(address(this))
        .feeNumerator();
    uint256 feeDenominator = IPeriodicPCOParamsReadable(address(this))
        .feeDenominator();

    return (bidAmount * feeNumerator) / feeDenominator;
}
```

For any auction which intends on collecting fees on behalf of the creator circle*, the [`bidAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L289C17-L289C26) specified by the caller **must be artificially inflated** relative to the [`collateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L290C17-L290C33) in order for the strict equality check to be satisfied.

> _* due to low severity implementation quirks this applies to *all* auctions_

### The Exploit

With that initial context out of the way, let's get down to the main issue.

Let's assume that the incumbent steward intends to submit a [`collateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L290C17-L290C33) of `1 ether` and the periodic honorarium is configured to 10%. This will necessitate a corresponding [`bidAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L289C17-L289C26) of `1.1 ether`, and save a [`Bid`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/IEnglishPeriodicAuctionInternal.sol#L8C12-L8C15) as follows:

```solidity
bid.bidder = incumbentStewardAddress;
bid.bidAmount = 1.1 ether;
bid.feeAmount = 1 ether;
bid.collateralAmount = 1 ether;
```

Let's fast forward to the end of the auction, and the incumbent steward is the winner.

Before settling the auction via the call to [`_closeAuction(uint256)`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L465C14-L465C44), the incumbent steward transfers the token to another incumbent-controlled address, `0xdeadbeef`.

Let's step through what happens.

1. Firstly, the `oldBidder` variable is assigned to `0xdeadbeef`, and not the incumbent steward themselves.

```solidity
address oldBidder;

if (IStewardLicense(address(this)).exists(tokenId)) {
  oldBidder = IStewardLicense(address(this)).ownerOf(tokenId); /// 0xdeadbeef
}
```

2. Next, because the incumbent steward is not `0xdeadbeef`, we execute the following logic:

```solidity
if (l.highestBids[tokenId][currentAuctionRound].bidder != oldBidder) {
  // Transfer bid to previous bidder's collateral
@> l.availableCollateral[oldBidder] += l.highestBids[tokenId][currentAuctionRound].bidAmount;
   l.highestBids[tokenId][currentAuctionRound].collateralAmount = 0;
   l.bids[tokenId][currentAuctionRound][
     l.highestBids[tokenId][currentAuctionRound].bidder
   ].collateralAmount = 0;
}
```

> Here, we demonstrably transfer `1.1 ether` of `bidAmount` to `0xdeadbeef`, the incumbent-controlled address.
>
> Remember, we initially only deposited `1 ether`!

3. Next, we give the incumbent steward back control of the auctioned token by transferring it back from `0xdeadbeef`:

```solidity
// Transfer to highest bidder
IStewardLicense(address(this)).triggerTransfer(
  oldBidder, /// @audit 0xdeadbeef
  l.highestBids[tokenId][currentAuctionRound].bidder, // @audit incumbant_steward
  tokenId
);
```

4. Finally, we distribute the `feeAmount` to the beneficiary - that's right, everyone gets paid! 🤑

```solidity
// Distribute fee to beneficiary
if (l.highestBids[tokenId][currentAuctionRound].feeAmount > 0) {
  IBeneficiary(address(this)).distribute{
    value: l.highestBids[tokenId][currentAuctionRound].feeAmount
}();
}
```

The incumbent steward gets to hold onto their token, and make a profit of `0.1 ether` for the trouble.

## Impact

This exploit enables the incumbent steward to permanently hold the asset risk-free and make profit on the asset in proportion to the periodic honorarium fee on their [`totalCollateralAmount`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L315C17-L315C38), an attacker-controlled unit, at the expense of honest bidders.

High.

## Code Snippet

### 📄 [EnglishPeriodicAuctionInternal.sol](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/IEnglishPeriodicAuctionInternal.sol)

## Tool used

Manual Review

## Recommendation

Don't give the incumbent steward a privileged path of execution.