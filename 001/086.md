Jovial Goldenrod Dachshund

high

# user can withdraw more than they deposited

## Summary
invariant test break when other users withdraw funds and the contract has fewer funds.  Every time a user cancels a bid they 
entered for their collateral increases  to a value grater than what they deposited and the same collateral value is used when
 they make a withdraw.

## Vulnerability Detail

1.  in this line of code  `bidAmount` can be set to be greater than the `msg.value` by a user  when placing a bid

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/facets/EnglishPeriodicAuctionFacet.sol#L167

```solidity 
    _placeBid(tokenId, msg.sender, bidAmount, msg.value);
```
2.when the user cancels  the bid this  will increase the user total  Collateral that they can withdraw

```solidity
 l.availableCollateral[bidder] += bid.collateralAmount;
```
https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L406


3. a user can repeat this until the auction ends and then call withdraw and drain the funds


## Impact
User can withdraw more than they deposited  and steal funds and cause a Denial of service for other users who want to withdraw
their funds from the contract. can be used to steal funds and drain contract

## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/facets/EnglishPeriodicAuctionFacet.sol#L153


https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/facets/EnglishPeriodicAuctionFacet.sol#L167

## Tool used

Manual Review , slither ,fuzz testing

## Recommendation
1. subtract `bidamout` from `msg.value` when making a bid and when user cancels bid add it back to the collateral


2.add this to the to code 

```diff
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
+          if(bidAmount> msg.value) revert();

             _placeBid(tokenId, msg.sender,  bidAmount, msg.value);
    }
     
```
