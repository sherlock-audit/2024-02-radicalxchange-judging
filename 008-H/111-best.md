Electric Champagne Mole

medium

# `feeAmount` can get locked because of missing returned value check after distributing `SETH` to beneficiaries

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