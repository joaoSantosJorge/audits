### [H-06] User sending ether to the contract is not registered as participant

**Description:** 

`ChristmasDinner::receive()` does not register the address of the doner as a participant breaking the contract promise.

```javascript
    receive() external payable {
@>      etherBalance[msg.sender] += msg.value;
        emit NewSignup(msg.sender, msg.value, true);
    }
```

**Impact:**

Impact: Hight. Funds are sent to the contract and the user will not be registered. It breaks the contract logic.

Likelyhood: High. If a user sends only ether to the contract he will not be registered for the dinner and will not be able to attend.

**Proof of Concept:**

```javascript
    function test_depositEtherAndBecomeParticipant() public {
        address payable _cd = payable(address(cd));
        vm.deal(user1, 10e18);
        vm.prank(user1);
        (bool sent, ) = _cd.call{value: 1e18}("");
        require(sent, "transfer failed");
        assertEq(user1.balance, 9e18);
        assertEq(address(cd).balance, 1e18);
        assertEq(cd.getParticipationStatus(user1), true);
    }
```
user1 is not on the participant list after sending ether to the contract.

**Recommended Mitigation:**

add the address to participation list.

```diff
    receive() external payable {
+       participant[msg.sender] = true;
        etherBalance[msg.sender] += msg.value;
        emit NewSignup(msg.sender, msg.value, true);
    }
```