### [L-04] `deadline` once set by the host can change again

**Description:** 

The boolean variable `deadlineSet` is never activated once the host defines the deadline for sign ups. Therefore, the host is able to change the deadline multiple times.

```javascript
    function setDeadline(uint256 _days) external onlyHost {
        if (deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
@>          deadline = block.timestamp + _days * 1 days;
            emit DeadlineSet(deadline);
        }
    }
```

**Impact:**

Impact: Low. No funds lost and the functionality is still the same since only the host can change the deadline.

Likelyhood: Low. The host can change the deadline forward and backwards, but the host have no reason to sabotage himself.

**Proof of Concept:**

```javascript
    function test_deadlineAlreadySet() public {
        vm.startPrank(deployer);
        cd.setDeadline(1 days);
        uint256 deadline1 = cd.deadline();
        cd.setDeadline(2 days);
        uint256 deadline2 = cd.deadline();
        vm.stopPrank();
        assertEq(deadline1, deadline2);
    }
```
The host is able to change the deadline even though it was already set before.

**Recommended Mitigation:**
Update the variable `deadlineSet` once the deadline is chaged by the host.

```diff
    function setDeadline(uint256 _days) external onlyHost {
        if (deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
            deadline = block.timestamp + _days * 1 days;
+           deadlineSet = true;
            emit DeadlineSet(deadline);
        }
    }
```