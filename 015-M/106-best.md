Mean Seafoam Dachshund

high

# [H-02] Current steward has too much advantage in the auction leading to multiple problems.

## Summary

Due to the collateral calculation in `_placeBid()` function, it is nearly impossible for a new bidder to win the auction over an existing steward. This will discourage any new bidders to enter the auction, allowing the existing steward to pay only the minimum fee amount to keep the Steward License token. This stagnation results in minimal fee collection and stops the auction from being competitive.
Another aspect of this is that current steward being able to pay very little amounts of collateral to drive the highest bid in the auction up arbitrarily making the price inflate.

## Vulnerability Detail

When a new user wants to become the steward of the Steward License token, they must win the auction against other bidders. including the current steward (mentioned as `currentBidder` in `_placeBid()`). However, due to collateral calculations for the `currentBidder` at https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L300-L303 :

```solidity
        if (bidder == currentBidder) {
            // If current bidder, collateral is entire fee amount
            feeAmount = totalCollateralAmount;
```

`currentBidder` only needs to deposit the fee amount as their collateral. This makes it nearly impossible for a new bidder to win considering they need to send more than 10x the amount of collateral to match `currentBidder`.

Here's an example scenario:
(Fee percentage is the same as the developer team's tests.)

- User1 becomes the `currentBidder` due to being named the `initialBidder` in contract creation.
- When the new round of auction starts, User1 can bid 10.1 ether while only sending 1.01 ether as collateral due to the check in the code snippet above.
- For User2 to match this offer, they would need to bid 10.2 ether and deposit 11.22 ether as collateral, meaning more than 10x the amount User1 would need to deposit in total. This makes it nearly impossible for any user to match User1's bid.

## PoC

```tsx
    it.only('new bidder cannot win auction', async function () {
      // Auction start: Now - 200
      // Auction end: Now + 100
      const instance = await getInstance({
        auctionLengthSeconds: 300,
        initialPeriodStartTime: (await time.latest()) - 200,
        licensePeriod: 1000,
      });
      //act as round 0 to make bidder2 the initial currentBidder.
      const bidAmount2 = ethers.utils.parseEther('1.2');
      const feeAmount2 = await instance.calculateFeeFromBid(bidAmount2);
      const collateralAmount2 = feeAmount2.add(bidAmount2);
      await instance
        .connect(bidder2)
        .placeBid(0, bidAmount2, { value: collateralAmount2 });

      await time.increase(100);
      //close round 0 auction.
      await instance.connect(bidder2).closeAuction(0);

      await time.increase(1100);
      //amount of bids for round 1.
      const bidAmount3 = ethers.utils.parseEther('10.1');
      const feeAmount3 = await instance.calculateFeeFromBid(bidAmount3);
      const collateralAmount3 = feeAmount3;

      const bidAmount4 = ethers.utils.parseEther('10.0');
      const feeAmount4 = await instance.calculateFeeFromBid(bidAmount4);
      const collateralAmount4 = feeAmount4.add(bidAmount4);
      //print amount bid, fee and collateral paid by users first log is the currentBidder(Steward). 
      console.log("bid amount:", ethers.utils.formatEther(bidAmount3), "fee amount:", ethers.utils.formatEther(feeAmount3), "collateral amount:", ethers.utils.formatEther(collateralAmount3));
      console.log("bid amount:", ethers.utils.formatEther(bidAmount4), "fee amount:", ethers.utils.formatEther(feeAmount4), "collateral amount:", ethers.utils.formatEther(collateralAmount4));
      //try to bid with both users
      await instance
        .connect(bidder2)
        .placeBid(0, bidAmount3, { value: collateralAmount3 });

      await instance
        .connect(bidder1)
        .placeBid(0, bidAmount4, { value: collateralAmount4 });
    });
```

Test will print the following logs:

```jsx
bid amount: 10.1 fee amount: 1.01 collateral amount: 1.01
bid amount: 10.0 fee amount: 1.0 collateral amount: 11.0
```

And revert with the following error: 

```jsx
Error: VM Exception while processing transaction: reverted with reason string 'EnglishPeriodicAuction: Bid amount must be greater than highest outstanding bid'
```

This test shows that bidder2 sent 1.01 ether as collateral while bidder1 sent 11.0 ethers and bidder1 still can not outbid bidder2. In order for bidder1 to win this auction they would need to send 11.22 ether as collateral compared to 1.01 ether sent by bidder2 (`currentBidder`).

## Impact

This vulnerability makes it near impossible for any new bidder to win the auctions over the `currentBidder` (current owner of the Steward License token). Since the discrepancy in the amount of collateral sent is so high between `currentBidder` and a new bidder, new bidders have no incentive to join the auctions as it is not possible for them to win. As the only bidder, `currentBidder` can keep bidding the `l.startingBid` and only pay the minimum amount of fee to keep the Steward License token. Even if any new bidder bids in the auction, `currentBidder` only needs to pay 10% of the collateral amount sent by the new bidder to win the auction again. This situation will discourage any new bidders from joining the auction, making the license only earn the minimum amount of fee.
Another aspect of this impact is `currentBidder` can deposit 0.1 ether of collateral to drive the highest bid up by 1+ ether. This will lead to inflation and manipulation of the highest bid.

## Code Snippet

https://github.com/sherlock-audit/2024-02-radicalxchange/blob/459dfbfa73f74ed3422e894f6ff5fe2bbed146dd/pco-art/contracts/auction/EnglishPeriodicAuctionInternal.sol#L286-L373

## Tool used

Manual Review, Hardhat

## Recommendation

Make sure that `currentBidder` need to match the collateral amount of highestBid - amount of collateral they have paid so far. This will create a more even auction and not allow the `currentBidder` to have a monopoly on the license or to arbitrarily inflate highestBid by sending low amounts of collaterals.
