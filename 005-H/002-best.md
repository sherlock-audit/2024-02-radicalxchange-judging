Curly Hemp Alligator

high

# Incorrect implementation of _isAuctionPeriod will cause _closeAuction to revert every time.

## Summary

The  `isAuctionPeriod` will return true once block.time > _auctionStartTime but will continue to return true even when the auctionEndTime has passed. 

This has critical impact since the `triggerTransfer` function in  `closeAuction` calls  `_transfer`, which has a check in the  `_beforeTokenTransfer` which reverts when  `isAuctionPeriod` returns true. This was set by the protocol to block license transfers during auctions. 

As a consequence, every  `closeAuction` call will revert with the message  `'StewardLicenseFacet: Cannot transfer during auction period'` and the protocol is permanently bricked.       

## Vulnerability Detail

First, let's take a look at `_isAuctionPeriod`.

```solidity
    function _isAuctionPeriod(uint256 tokenId) internal view returns (bool) {
        if (tokenId >= IStewardLicense(address(this)).maxTokenCount()) {
            return false;
        }
        //slither-disable-next-line timestamp
        return block.timestamp >= _auctionStartTime(tokenId);
    }
```
The function should return `true` when the auction is ongoing and `false` when it is not yet. Yet the code only checks that `block.timestamp` is greater then the auction start time. It does not check that the `block.timestamp` has exceeded the auction end time. 

As a result, once the auction has started, this function will always return `true`, regardless of the actual status of the auction.    

Why is this important? Because the protocol implemented a check to make sure the license could not change ownership **during** auctions. 

In `EnglishPeriodicAuctionInternal: _closeAuction`:
```solidity
        IStewardLicense(address(this)).triggerTransfer(
            oldBidder,
            l.highestBids[tokenId][currentAuctionRound].bidder,
            tokenId
        );
```
In `StewardLicenseBase: triggerTransfer`:
```solidity
    function triggerTransfer( address from, address to, uint256 tokenId) external {
        require(
            msg.sender == address(this),
            'NativeStewardLicense: Trigger transfer can only be called from another facet'
        );

        _triggerTransfer(from, to, tokenId);
    }
```
In `StewardLicenseInternal: _triggerTransfer`:
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
In `ERC721BaseInternal: _transfer`:
```solidity
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {
        address owner = _ownerOf(tokenId);

        if (owner != from) revert ERC721Base__NotTokenOwner();
        if (to == address(0)) revert ERC721Base__TransferToZeroAddress();

        _beforeTokenTransfer(from, to, tokenId);

        ERC721BaseStorage.Layout storage l = ERC721BaseStorage.layout();

        l.holderTokens[from].remove(tokenId);
        l.holderTokens[to].add(tokenId);
        l.tokenOwners.set(tokenId, to);
        l.tokenApprovals[tokenId] = address(0);

        emit Approval(owner, address(0), tokenId);
        emit Transfer(from, to, tokenId);
    }
```
And finally, we find the check in `StewardLicenseInternal: _beforeTokenTransfer`:
```solidity
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual override(ERC721BaseInternal, ERC721Metadata) {
        // Disable transfers if not mint
        if (from != address(0x0)) {
            // External call is to known contract
            //slither-disable-next-line calls-loop
            bool isAuctionPeriod = IPeriodicAuctionReadable(address(this))
                .isAuctionPeriod(tokenId);
            require(
                !isAuctionPeriod,
                'StewardLicenseFacet: Cannot transfer during auction period'
            );
        }

        super._beforeTokenTransfer(from, to, tokenId);
    }
```
Since the `isAuctionPeriod` will always be true when calling `_closeAuction`, the check will resolve to false and the function will revert with the message `'StewardLicenseFacet: Cannot transfer during auction period'`.

## Impact

The impact is critical since no auctions can ever be closed so the licenses remain in limbo for perpetuity. 

## Code Snippet


https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L224-L230

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L465-L538

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseBase.sol#L15-L27

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseInternal.sol#L96-L108

@node_modules/@solidstate/contracts/token/ERC721/base/ERC721BaseInternal.sol#L109-L130

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/main/pco-art/contracts/license/StewardLicenseInternal.sol#L206-L224

## Tool used

Manual Review

## Recommendation

Change the logic of the function so that it will return `false` once the auction end time has been reached. 
```diff
    function _isAuctionPeriod(uint256 tokenId) internal view returns (bool) {
        if (tokenId >= IStewardLicense(address(this)).maxTokenCount()) {
            return false;
        }
-        return block.timestamp >= _auctionStartTime(tokenId);
+       return block.timestamp >= _auctionStartTime(tokenId) && block.timestamp <= _auctionEndTime(tokenId);
```
