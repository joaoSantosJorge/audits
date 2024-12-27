### [M-01] Contract Insuficient Funds Check Missing

**Description:** 
When a player wins, the contract may have insufficient funds.
There is no check in the beginning of the game to see if the contract has enough eth to send to the player in case he wins.

```javascript
    function startGame() public payable returns (uint256) {
@>      //missing check
        address player = msg.sender;
        require(msg.value >= 1 ether, "not enough ether sent");
        initializeDeck(player);
        uint256 card1 = drawCard(player);
        uint256 card2 = drawCard(player);
        addCardForPlayer(player, card1);
        addCardForPlayer(player, card2);
        return playersHand(player);
    }
```


**Impact:**
Likelyhood: Medium, at some point it is possible that the contract runs out of ether, even though `TwentyOne.s.sol` script file sugests the contract would start with 10 ether.
Impact: Medium, The promised eth reward is not sent since the protocol has no eth.

**Proof of Concept:**
Since there is no contract eth balance and we force the player to win against the dealer, the `TwentyOne::endGame()` will revert because the contract has insuficient eth.
```javascript
    function test_No_Contract_Supply() public {
        vm.startPrank(player1); // Start acting as player1

        twentyOne.startGame{value: 1 ether}();

        // Mock the dealer's behavior to ensure player wins
        // Simulate dealer cards by manipulating state
        vm.mockCall(
            address(twentyOne),
            abi.encodeWithSignature("dealersHand(address)", player1),
            abi.encode(18) // Dealer's hand total is 18
        );

        //here, the revert is expected since there is no ether in twentyOne contract. The player will not receive ether.
        vm.expectRevert();
        twentyOne.call();

        vm.stopPrank();
    }
```

**Recommended Mitigation:**
When starting a game, make sure the contract has enough funds.
To allow multiple games to run simultaneously, add the variable `fundsAtRisk`
```diff
    mapping(address => PlayersCards) playersDeck;
    mapping(address => DealersCards) dealersDeck;
    mapping(address => uint256[]) private availableCards;
+   uint256 private fundsAtRisk;        //stores the amount of funds the contract can lose
```
add the check when starting the game:
```diff
function startGame() public payable returns (uint256) {
+   fundsAtRisk = fundsAtRisk + 1;
+   require(address(this).balance >= fundsAtRisk, "Contract Insuficient Funds");
    address player = msg.sender;
    require(msg.value >= 1 ether, "not enough ether sent");
    initializeDeck(player);
    uint256 card1 = drawCard(player);
    uint256 card2 = drawCard(player);
    addCardForPlayer(player, card1);
    addCardForPlayer(player, card2);
    return playersHand(player);
}
```
once the game ends, make sure the `fundsAtRisk` is updated
```diff
function endGame(address player, bool playerWon) internal {
    delete playersDeck[player].playersCards; // Clear the player's cards
    delete dealersDeck[player].dealersCards; // Clear the dealer's cards
    delete availableCards[player]; // Reset the deck
    if (playerWon) {
        payable(player).transfer(2 ether); // Transfer the prize to the player
        emit FeeWithdrawn(player, 2 ether); // Emit the prize withdrawal event
    }
+   fundsAtRisk = fundsAtRisk - 1;
}
```
Note: Check the rest of the contract to update the `fundsAtRisk` case needed.

**Tools Used:**
Manual Review

**Additional Note to dev**
The unit test `test_Call_PlayerWins` also fails because contract has no ether. For it to pass, add the following line to `TwentyOneTest::test_Call_PlayerWins()`

```diff
    function test_Call_PlayerWins() public {
+       vm.deal(address(twentyOne), 10 ether);
        vm.startPrank(player1); // Start acting as player1

        twentyOne.startGame{value: 1 ether}();

        // Mock the dealer's behavior to ensure player wins
        // Simulate dealer cards by manipulating state
        vm.mockCall(
            address(twentyOne),
            abi.encodeWithSignature("dealersHand(address)", player1),
            abi.encode(18) // Dealer's hand total is 18
        );

        uint256 initialPlayerBalance = player1.balance;

        // Player calls to compare hands
        twentyOne.call();

        // Check if the player's balance increased (prize payout)
        uint256 finalPlayerBalance = player1.balance;
        assertGt(finalPlayerBalance, initialPlayerBalance);

        vm.stopPrank();
    }
```