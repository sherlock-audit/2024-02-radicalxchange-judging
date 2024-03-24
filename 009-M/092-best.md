High Sage Skunk

medium

# `EnglishPeriodicAuctionInternal::_placeBid` function may revert because of rounded fee calculation

## Summary
`EnglishPeriodicAuctionInternal::_placeBid` function compares `calculatedFeeAmount` and `feeAmount`. `calculatedFeeAmount` is calculated with `(bidAmount * feeNumerator) / feeDenominator;` formula and this may rounded and make `_placeBid` function revert. 

## Vulnerability Detail
1. Bidder calls the `EnglishPeriodicAuctionFacet::placeBid` function and this will redirect to `EnglishPeriodicAuctionInternal::_placeBid` function.
2. In [line 337-340](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L337-L340) there is a requirement and this redirect to `_checkBidAmount` function.
```solidity
require(
            _checkBidAmount(bidAmount, feeAmount),
            'EnglishPeriodicAuction: Incorrect bid amount'
        );
```
3. This function compares `calculatedFeeAmount` and `feeAmont` and then returns bool.
```solidity
function _checkBidAmount(
        uint256 bidAmount,
        uint256 feeAmount
    ) internal view returns (bool) {
        uint256 calculatedFeeAmount = _calculateFeeFromBid(bidAmount);

        return calculatedFeeAmount == feeAmount;
    }
```
4. `feeNumerator / feeDenominator` can be between 0-almost infinity. So there is no limitation for this numbers. If `feeDenominator` set as prime number it most likely be floating number. (Not just prime numbers. For example fee ratio is 1/6 and bid amount is 1,000,000,000,000 wei. `1.000.000.000.000/6 = 166.666.666.666,6667` but solidity will round this to `166.666.666.666`). With that `_checkBidAmount` function will return false almost all call. 
```solidity
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

Also if `bidAmount` ends not 0 like ".....1 wei", again it most probably will rounded in `_calculateFeeFromBid` function.

Side note: If `feeDenominator` is much higher than `bidAmount * feeNumerator` it will always round to 0.
## Impact
Bidders may not place their bid.

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L605-L615

## Tool used
Manual Review

## Recommendation
Make these changes in the code and if there is any remaining dust, refund it to the bidder.
```diff
function _checkBidAmount(
        uint256 bidAmount,
        uint256 feeAmount
    ) internal view returns (bool) {
        uint256 calculatedFeeAmount = _calculateFeeFromBid(bidAmount);

-       return calculatedFeeAmount == feeAmount;
+       return calculatedFeeAmount <= feeAmount;
    }
```