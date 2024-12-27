### [M-05] Host can withdraw funds before deadline

**Description:** 

`ChristmasDinner::withdraw()` should only be possible after deadline has passed. No such verification is made.

```javascript
    function withdraw() external onlyHost {
@>      address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
    }
```

**Impact:**

Impact: Medium. Funds can be retrieved earlier even though only by the host.

**Proof of Concept:**

```javascript
    function test_withdrawAsHostBeforeDeadline() public {
        uint256 wbtcAmount;
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 0.5e18);
        wbtcAmount += 0.5e18;
        vm.stopPrank();
        vm.startPrank(deployer);
        cd.setDeadline(5);
        cd.withdraw();
        assertEq(wbtc.balanceOf(deployer), 0);
    }
```
The host is able to withdraw funds before the deadline.

**Recommended Mitigation:**

Do the necessary verification step.

```diff
    function withdraw() external onlyHost {
+       require(block.timestamp > deadline, "Event not over yet");
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
    }
```