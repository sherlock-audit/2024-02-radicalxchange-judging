# Issue H-1: Highest bidder can withdraw his collateral due to a missing check in _cancelAllBids 

Source: https://github.com/sherlock-audit/2024-02-radicalxchange-judging/issues/14 

## Found by 
0rpse, 0xKartikgiri00, 0xPwned, 0xShitgem, 0xboriskataa, 0xbrivan, 14si2o\_Flint, 404666, AMOW, Aamirusmani1552, AgileJune, Al-Qa-qa, Atharv, CarlosAlbaWork, DMoore, DenTonylifer, Dots, FSchmoede, FassiSecurity, FastTiger, Krace, Marcologonz, SovaSlava, Tendency, Tricko, aycozynfada, cats, cawfree, cocacola, cu5t0mPe0, dipp, ethernal, fugazzi, ge6a, jah, jasonxiale, ke1caM, koreanspicygarlic, kuprum, ljj, merlin, mrBmbastic, neocrao, neon2835, offside0011, psb01, pseudoArtist, pynschon, sammy, sandy, tedox, thank\_you, theFirstElder, theOwl, thisvishalsingh, turvec, valentin2304, zraxx, zzykxx
## Summary

A bidder with the highest bid cannot cancel his bid since this would break the auction. A check to ensure this was implemented in `_cancelBid`.

However, this check was not implemented in `_cancelAllBids`, allowing the highest bidder to withdraw his collateral and win the auction for free.  

## Vulnerability Detail

The highest bidder should not be able to cancel his bid, since this would break the entire auction mechanism. 

In `_cancelBid` we can find a require check that ensures this:

```solidity
        require(
            bidder != l.highestBids[tokenId][round].bidder,
            'EnglishPeriodicAuction: Cannot cancel bid if highest bidder'
        );

```
Yet in `_cancelAllBids`, this check was not implemented. 
```solidity
     * @notice Cancel bids for all rounds
     */
    function _cancelAllBids(uint256 tokenId, address bidder) internal {
        EnglishPeriodicAuctionStorage.Layout
            storage l = EnglishPeriodicAuctionStorage.layout();

        uint256 currentAuctionRound = l.currentAuctionRound[tokenId];

        for (uint256 i = 0; i <= currentAuctionRound; i++) {
            Bid storage bid = l.bids[tokenId][i][bidder];

            if (bid.collateralAmount > 0) {
                // Make collateral available to withdraw
                l.availableCollateral[bidder] += bid.collateralAmount;

                // Reset collateral and bid
                bid.collateralAmount = 0;
                bid.bidAmount = 0;
            }
        }
    }

```
Example: 
User Bob bids 10 eth and takes the highest bidder spot. 
Bob calls `cancelAllBidsAndWithdrawCollateral`.

The `_cancelAllBids` function is called and this makes all the collateral from all his bids from every round available to Bob. This includes the current round `<=` and does not check if Bob is the current highest bidder. Nor is `l.highestBids[tokenId][round].bidder` reset, so the system still has Bob as the highest bidder. 

Then `_withdrawCollateral` is automatically called and Bob receives his 10 eth  back. 

The auction ends. If Bob is still the highest bidder, he wins the auction and his bidAmount of 10 eth is added to the availableCollateral of the oldBidder. 

If there currently is more than 10 eth in the contract (ongoing auctions, bids that have not withdrawn), then the oldBidder can withdraw 10 eth. But this means that in the future a withdraw will fail due to this missing 10 eth. 

## Impact

A malicious user can win an auction for free. 

Additionally, either the oldBidder or some other user in the future will suffer the loss.  

If this is repeated multiple times, it will drain the contract balance and all users will lose their locked collateral. 

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L416-L436

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L380-L413

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L468-L536

## Tool used

Manual Review

## Recommendation

Implement the require check from _cancelBid to _cancelAllBids.

# Issue H-2: `feeAmount` can get locked because of missing returned value check after distributing `SETH` to beneficiaries 

Source: https://github.com/sherlock-audit/2024-02-radicalxchange-judging/issues/111 

## Found by 
Al-Qa-qa, zzykxx
## Summary
Because of missing the returned value when distributing `feeAmount` to beneficiaries, which is a `bool` value similar to `ERC20` transfer. The `feeAmount` can get locked if the distributing function fails.

## Vulnerability Detail

1. When closing the Auction, the `feeAmount` is distributed to beneficiaries in the last of the function. But without checking if the function succeeded or not.

[auction/EnglishPeriodicAuctionInternal.sol#L534-L536](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L534-L536)
```solidity
    function _closeAuction(uint256 tokenId) internal {
        ...

        // Distribute fee to beneficiary
        if (l.highestBids[tokenId][currentAuctionRound].feeAmount > 0) {
<@          IBeneficiary(address(this)).distribute{
                value: l.highestBids[tokenId][currentAuctionRound].feeAmount
            }();
        }
    }
```

2. This function calls `IDABeneficiaryFacet.distribute()`

[beneficiary/facets/IDABeneficiaryFacet.sol#L69-L76](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/beneficiary/facets/IDABeneficiaryFacet.sol#L69-L76)
```solidity
    function distribute() external payable {
        require(
            msg.value > 0,
            'IDABeneficiaryFacet: msg.value should be greater than 0'
        );

        _distribute(msg.value);
    }
```

3. which calls `IDABeneficiaryInternal::_distribute()`

[beneficiary/IDABeneficiaryInternal.sol#L78-L89](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/beneficiary/IDABeneficiaryInternal.sol#L78-L89)

```solidity
    function _distribute(uint256 value) internal {
        IDABeneficiaryStorage.Layout storage l = IDABeneficiaryStorage.layout();

        emit Distributed(value);

        // Wrap ETH
        l.token.upgradeByETH{ value: value }();

        // Distribute to beneficiaries
        // @audit Missing Returned Value
<@      l.token.distribute(0, value);
    }
```

This function `l.token.distribute(0, value)` is a function in library `SuperTokenV1Library`, which is used to handle `SETH` token distributing to more than one beneficiary, and as we can see below it returned `true`, if distributing (transferring) of SETH done successfully.

[superfluid-finance::apps/SuperTokenV1Library.sol#L1327](https://github.com/superfluid-finance/protocol-monorepo/blob/dev/packages/ethereum-contracts/contracts/apps/SuperTokenV1Library.sol#L1327)
```solidity
    function distribute( ... ) internal returns (bool) {
        (ISuperfluid host, IInstantDistributionAgreementV1 ida) = _getAndCacheHostAndIDA(token);
        host.callAgreement( ... );

<@      return true;
    }
``` 

So if distributing tokens to the beneficiary fails, the Auction will not take this case into consideration, which will make `SETH` tokens locked in The contract.
 
## Impact
`feeAmount` getting locked in the NFT collection contract as `SETH` tokens 

## Code Snippet
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L534-L536

## Tool used
Manual Review

## Recommendation
- First, You need to return the bool value, which will determine if the distribution succeeded or failed, from the `IDABeneficiaryFacet.distribute()`. 

- Then, Check its value in `EnglishPeriodicAuctionInternal::_closeAuction()`.

```diff
    function _closeAuction(uint256 tokenId) internal {
        ...

        // Distribute fee to beneficiary
        if (l.highestBids[tokenId][currentAuctionRound].feeAmount > 0) {
-           IBeneficiary(address(this)).distribute{
+           bool success = IBeneficiary(address(this)).distribute{
                value: l.highestBids[tokenId][currentAuctionRound].feeAmount
            }();

+           require(success, "EnglishPeriodicAuction: Failed to distribute feeAmound to beneficiaries");
        }
    }
```

# Issue M-1: The incumbent steward can avoid paying the periodic honorarium. 

Source: https://github.com/sherlock-audit/2024-02-radicalxchange-judging/issues/9 

## Found by 
cawfree, zzykxx
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

# Issue M-2: The protocol is not compatible with collections of NFTs with non-sequential IDs or sequential IDs that don't start from 0 

Source: https://github.com/sherlock-audit/2024-02-radicalxchange-judging/issues/16 

## Found by 
zzykxx
## Summary
The protocol is not compatible with collections of NFTs with non-sequential IDs or sequential IDs that don't start from 0.

## Vulnerability Detail
The protocol allows to mint an auctionable NFT with a custom ID via [mintToken()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseBase.sol#L31) but the auction system assumes that the IDs of the auctioned NFTs are sequential and starting from 0.

This can cause issues in the following functions:
- [_isAuctionPeriod()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L224-L230), called to know if an NFT with ID `tokenId` can be auctioned, always returns `false` if the `tokenId` is lower than `maxTokenCount`. This means that NFTs minted with an ID bigger than the amount of NFTs currently available in the collection cannot be auctioned.
- [_isReadyForStranfer()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L568) performs the same check as [_isAuctionPeriod()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L224-L230).
- [_auctionStartTime()](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L543) can set the auction start time to [`initialPeriodStartTimeOffset * tokenId`](https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L568), which means an auction for an NFT with a very high ID can potentially be far in the future. 

## Impact
It might be impossible to auction some NFTs. 

## Code Snippet

## Tool used

Manual Review

## Recommendation
Adjust the protocol in such a way that NFTs with non-sequential IDs or IDs not starting from 0 can be used.

