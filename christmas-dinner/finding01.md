### [M-01] When a participant asks a full refund, the participant list should be updated accordingly.

**Description:**

If a participant changes plans and decides not to attend a dinner, he will ask a `ChristmasDinner::refund()` (before the deadline). The participant list should reflect the refund decision that implies the participant will not attend the dinner.

```javascript
    function refund() external nonReentrant beforeDeadline {
        address payable _to = payable(msg.sender);
        _refundERC20(_to);
        _refundETH(_to);
        emit Refunded(msg.sender);
@>        
    }
```
The function `ChristmasDinner::refund()` should update the participant attendance in the dinner.


**Impact:**

Likelyhood - High, it is normal for people to change opinions. To facilitate the organization of the dinner and its cost, the participant list should be as accurate as possible.

Impact - Medium/low, no funds are lost, its functionality is slightly affected.


**Proof of Concept:**

```javascript
    function test_refundAndNotParticipant() public {
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 1e18);
        assertEq(wbtc.balanceOf(address(cd)), 1e18);
        cd.refund();
        assertEq(wbtc.balanceOf(address(cd)), 0);
        assertEq(cd.getParticipationStatus(user1), false);
    }
```
The participant list should reflect the fact that `user1` whithdrew all the funds that confirmed his appearence in the dinner.
But the participant list still expects `user1` to attend the dinner.


**Recommended Mitigation:**

```diff
    function refund() external nonReentrant beforeDeadline {
        address payable _to = payable(msg.sender);
        _refundERC20(_to);
        _refundETH(_to);
        emit Refunded(msg.sender);
+       participant[msg.sender] = false;
    }
```

After a refund, the user in participant list should be set to `false`.