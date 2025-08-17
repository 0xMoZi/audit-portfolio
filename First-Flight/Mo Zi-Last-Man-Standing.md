# Last Man Standing - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Throne Can Be Claimed After Grace Period Ends](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Wrong require checks in `Game::claimThrone` function.](#M-01)
- ## Low Risk Findings
    - ### [L-01. Emiting zero prizeAmount/pot value when declare winner](#L-01)
    - ### [L-02. Game::getContractBalance() May Return Unexpected Value](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #45

### Dates: Jul 31st, 2025 - Aug 7th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-07-last-man-standing)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Throne Can Be Claimed After Grace Period Ends            



# Throne Can Be Claimed After Grace Period Ends

## Description

The `claimThrone()` function does not validate whether the grace period has ended before allowing users to claim the throne. This design flaw leads to two significant issues:

1. **Late Claims after Grace Period**: Players can continue to call `claimThrone()` even after the game should have ended, which resets the timer (`lastClaimTime`) and extends the game unfairly.
2. **Front-running Vulnerability**: Without a cutoff time, attackers can submit a transaction at the very end of the grace period (or front-run a visible transaction in the mempool) to become the last claimant and secure the throne unfairly.

Both issues undermine the fairness and finality of the game logic.

```Solidity
//@audit-issue Front-running
function claimThrone() external payable gameNotEnded nonReentrant {
    //@audit-issue allowing malicious actor to claim the throne right after grace period expires and before decalare winner function be call
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");

    // ... another logic
}
```

## Risk

**Likelihood**:

* Any user (or bot) can exploit this at will.

* Especially dangerous in a competitive or incentivized game (e.g., winner receives the pot).

* Mempool visibility allows opportunistic front-running.

**Impact**:

* The game can be **prolonged indefinitely**, blocking finalization and pot distribution.

* **Unfair final kingship** may be awarded to a front-runner, reducing game integrity.

* Undermines trust in fairness of winner selection.

## Proof of Concept

here's the test that shown malicious actor can Claim the Throne after the grace period has ended and reseting the grace period:

```solidity
function test_StillManageToClaimThroneAfterGracePeriodEnd() public {
    // Checks how long the grace period is
    console2.log("\n Grace period is: %s", game.gracePeriod());

    // 1. Player1 will become the winner
    console2.log("\n Player1 Claim the Throne");
    vm.startPrank(player1);
    game.claimThrone{value: game.claimFee()}();
    console2.log("\n Current King is: %s", game.currentKing());
    vm.stopPrank();

    // 2. get the remaining time till grace period ended
    uint256 getTheRemainingTimeAfterPlayer1Claim = game.getRemainingTime();
   console2.log("\n Time remaining until the grace period expires: %s", getTheRemainingTimeAfterPlayer1Claim);

    // 3. Simulate the grace period has ended
    vm.warp(block.timestamp + game.gracePeriod() + 1);

    // 4. get the remaining time after grace period end
    uint256 getTheRemainingTimeAfterGracePeriodEnd = game.getRemainingTime();
    console2.log("\n Time remaining when the grace period was expires: %s", getTheRemainingTimeAfterGracePeriodEnd);

    // 5. Malicious actor Claim the Throne right in time when grace period has ended and before anyone call declare winner
    console2.log("\n Malicious actor Claim the Throne before anyone call declare winner");
    vm.startPrank(maliciousActor);
    game.claimThrone{value: game.claimFee()}();
    console2.log("\n Current King is: %s", game.currentKing());
    vm.stopPrank();

    // 6. get the remaining time after malicious actor claim the throne
    uint256 getTheRemainingTimeAfterMaliciousActorClaim = game.getRemainingTime();
    assertEq(getTheRemainingTimeAfterMaliciousActorClaim, 1 days);
    console2.log("\n Time remaining after Malicious Actor Claim the Throne: ", getTheRemainingTimeAfterMaliciousActorClaim);
}
```

and the log of the test:

```solidity
Ran 1 test for test/Game.t.sol:GameTest
[PASS] test_StillManageToClaimThroneAfterGracePeriodEnd() (gas: 219707)
Logs:

 Grace period is: 86400

 Player1 Claim the Throne

 Current King is: 0x7026B763CBE7d4E72049EA67E89326432a50ef84

 Time remaining until the grace period expires: 86400

 Time remaining when the grace period was expires: 0

 Malicious actor Claim the Throne before anyone call declare winner

 Current King is: 0x195Ef46F233F37FF15b37c022c293753Dc04A8C3

 Time remaining after Malicious Actor Claim the Throne 86400
```

## Recommended Mitigation

Introduce a new dynamic variable `gameDeadline` to track the end of the grace period. Do **not** initialize the game timer (`lastClaimTime` or `gameDeadline`) in the constructor. Instead, start the game only upon the **first claim**.

This approach ensures:

* The game does not start running until there is actual user activity.

* Prevents the game from expiring before any participation.

* Supports a fair and competitive dynamic grace period.

```diff
+ uint256 public gameDeadline; // new dynamic variable

+ uint256 public constant EXTENSION_WINDOW = 5 minutes; // new variable
+ uint256 public constant EXTENSION_DURATION = 5 minutes; // new variable

function claimThrone() external payable gameNotEnded nonReentrant {
    // Start the game on first claim
+   if (gameDeadline == 0) {
+       lastClaimTime = block.timestamp;
+        gameDeadline = block.timestamp + GRACE_PERIOD;
+   }
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
    require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
    
    // Reject claims if grace period has ended
+   require(
+       block.timestamp < gameDeadline,
+       "Game: Grace period has ended. No more claims allowed."
+   );

    // ... another logic

    // Extend the game if claim is made near the end
+   if (block.timestamp + EXTENSION_WINDOW >= gameDeadline) {
+       gameDeadline += EXTENSION_DURATION;
+       emit GameExtended(gameDeadline);
+   }

    // ... another logic
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Wrong require checks in `Game::claimThrone` function.            



# Users will never be able to enter the game due to require checks and always be reverting

## Description

In expected behavior users can enter using `Game::claimThrone` function by sending the required claim fee, but in reality users will never be able to enter the game due to require checks in `#L188` that only require `currentKing` to enter.

```solidity
        //@audit-issue how user will be able to enter the game if the msg.sender need to be current king?
@>      require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
```

## Risk

**Likelihood**:

* This will occur everytime users call `Game::claimThrone` function and revert immediately

**Impact**:

* No one can entering the game

## Proof of Concept

Player1 try to claim the throne and become a king by sending the required claim fee or more

```solidity
function test_ClaimThroneAndBecomeKing() public {
    address whoIsCurrentKing = game.currentKing();
    assertEq(whoIsCurrentKing, address(0));
        
    vm.startPrank(player1);
    game.claimThrone{value: 2 ether}();
    vm.stopPrank();
}
```

but the tx will always revert, because require checks only require `currentKing` to enter

```Solidity
 [48482] GameTest::test_ClaimThroneAndBecomeKing()
    ├─ [2618] Game::currentKing() [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    ├─ [0] VM::assertEq(0x0000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000) [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84])
    │   └─ ← [Return]
    ├─ [27259] Game::claimThrone{value: 2000000000000000000}()
    │   └─ ← [Revert] Game: You are already the king. No need to re-claim.
    └─ ← [Revert] Game: You are already the king. No need to re-claim.
```

the revert say Player1 is already the king, but in reality `currentKing` still address(0).

## Recommended Mitigation

```diff
- require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
+ require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
```

Now users can enter the game by claim the throne and become king replace prev king and replaced king can compete again till grace period end.


# Low Risk Findings         

## <a id='L-01'></a>L-01. Emiting zero prizeAmount/pot value when declare winner            



# Emiting zero prizeAmount/pot value when declare winner

## Description

In expected behavior prizeAmount/pot value in emiting event will reveal `The total prize amount won`, but in the function `Game::declareWinner` emit prizeAmount/pot didn't work as expected, the issue is in the `#L241`.

```solidity
function declareWinner() external gameNotEnded {
    require(currentKing != address(0), "Game: No one has claimed the throne yet.");
    require(
        block.timestamp > lastClaimTime + gracePeriod,
        "Game: Grace period has not expired yet."
    );

    gameEnded = true;

    pendingWinnings[currentKing] = pendingWinnings[currentKing] + pot;
@>    pot = 0; // Reset pot after assigning to winner's pending winnings

    emit GameEnded(currentKing, pot, block.timestamp, gameRound);
}
```

## Risk

**Likelihood**:

* This issue will occur when grace period end and when someone call `Game::declareWinner` to declare the KING

**Impact**:

* Tracking Errors and Transparency

* Integration Error with Off-chain Service

## Proof of Concept

Here's the test by emiting expected event

```solidity
function test_DeclareWinnerAndEmitGameEnded() public {
    vm.startPrank(player1);

    // 1. Player claim the throne
    game.claimThrone{value: game.claimFee()}();

    // 2. Pass grace period
    uint256 gracePeriodEnd = block.timestamp + GRACE_PERIOD + 1;
    vm.warp(gracePeriodEnd);

    // 3. Declare Player1 as a winner and emiting expected event GameEnded
    vm.expectEmit(true, false, false, true);
    emit GameEnded(player1, game.pot(), gracePeriodEnd, 1);
    game.declareWinner();

    vm.stopPrank();
}
```

and this the log from the test:

```solidity
[227539] GameTest::test_DeclareWinnerAndEmitGameEnded()
    ├─ [0] VM::startPrank(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84])
    │   └─ ← [Return]
    ├─ [2514] Game::claimFee() [staticcall]
    │   └─ ← [Return] 100000000000000000 [1e17]
    ├─ [150621] Game::claimThrone{value: 100000000000000000}()
    │   ├─ emit ThroneClaimed(newKing: player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84], claimAmount: 100000000000000000 [1e17], newClaimFee: 110000000000000000 [1.1e17], newPot: 95000000000000000 [9.5e16], timestamp: 1)
    │   └─ ← [Stop]
    ├─ [0] VM::warp(86402 [8.64e4])
    │   └─ ← [Return]
    ├─ [0] VM::expectEmit(true, false, false, true)
    │   └─ ← [Return]
    ├─ [515] Game::pot() [staticcall]
    │   └─ ← [Return] 95000000000000000 [9.5e16]
    ├─ emit GameEnded(winner: player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84], prizeAmount: 95000000000000000 [9.5e16], timestamp: 86402 [8.64e4], round: 1)
    ├─ [50689] Game::declareWinner()
    │   ├─ emit GameEnded(winner: player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84], prizeAmount: 0, timestamp: 86402 [8.64e4], round: 1)
    │   └─ ← [Stop]
    └─ ← [Revert] log != expected log
```

as you can see there's missmatch from expected event prizeAmount and the actual happen in `Game::declareWinner` function.

## Recommended Mitigation

```diff
function declareWinner() external gameNotEnded {
    require(currentKing != address(0), "Game: No one has claimed the throne yet.");
    require(
        block.timestamp > lastClaimTime + gracePeriod,
        "Game: Grace period has not expired yet."
    );

    gameEnded = true;

    pendingWinnings[currentKing] = pendingWinnings[currentKing] + pot;
+    emit GameEnded(currentKing, pot, block.timestamp, gameRound);
-    pot = 0; // Reset pot after assigning to winner's pending winnings

-    emit GameEnded(currentKing, pot, block.timestamp, gameRound);
+    pot = 0; // Reset pot after assigning to winner's pending winnings
}
```

place updated pot after emiting the event.

## <a id='L-02'></a>L-02. Game::getContractBalance() May Return Unexpected Value            



# Contract allows anyone to send Ether directly to the contract causing Game::getContractBalance() Return Unexpected Value

## Description

The `Game::getContractBalance()` function returns the raw Ether balance of the contract, it assumes that all ETH in the contract came through controlled functions and that the balance represents a valid sum of tracked internal state (like pot and platformfees).

```solidity
/**
 * @dev Returns the current balance of the contract (should match the pot plus platform fees unless payouts are pending).
 */
function getContractBalance() public view returns (uint256) {
@>    return address(this).balance; // returns raw ether balance
}
```

However, the contract includes a fallback receive() function:

```solidity
receive() external payable {}
```

This allows anyone to send Ether directly to the contract without calling a tracked logic function. As a result, address(this).balance can become higher than the expected return.

## Risk

**Likelihood**:

* when malicious user sends ETH directly to the contract via `send()` or `transfer()`.

**Impact**:

* shown misleading data

## Recommended Mitigation

Change getContractBalance() to return internal variable:

```solidity
function getContractBalance() public view returns (uint256) {
    return pot + platformFees; // or another internal sum
}
```



