### [H-07] Host does not receive ether when calls `ChristmasDinner::withdraw()`

**Description:** 

`ChristmasDinner::withdraw()` does not send ether to the host.

```javascript
    function withdraw() external onlyHost {
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
    }
```

**Impact:**

Impact: High. Funds that belong to the host stay in the contract.

Likelyhood: High. The host will not receive any eth funds that are sent to the contract.

**Proof of Concept:**

```javascript
    function test_withdrawEtherAsHost() public {
        address payable _cd = payable(address(cd));
        vm.deal(user1, 10e18);
        vm.prank(user1);
        (bool sent, ) = _cd.call{value: 1e18}("");
        require(sent, "transfer failed");
        assertEq(user1.balance, 9e18);
        assertEq(address(cd).balance, 1e18);
        vm.prank(deployer);
        cd.withdraw();
        assertEq(deployer.balance, 1e18);
    }
```
deployer does not receive ether sent by user1 to the contract. Deployer called `ChristmasDinner::withdraw()` when the contract had 1 eth locked.

**Recommended Mitigation:**

add the necessary logic to send ether to host.

```diff
    function withdraw() external onlyHost {
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));

+       (bool sent, ) = _host.call{value: address(this).balance}("");
+       require(sent, "Failed to send Ether");
    }
```