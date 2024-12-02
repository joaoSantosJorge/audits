### [H-02] `CharityRegistry::isVerified()` incorrectly checks if a charity was verified by the admin making it possible to donate to a charity that was not approved by the admin

**Description:** 
The function `CharityRegistry::isVerified()` instead of checking the `verifiedCharities` map, checks if a charity was registered or not. This makes it possible for a donation to happen to a charity that was not approved by the admin.

```javascript
function isVerified(address charity) public view returns (bool) {
@>    return registeredCharities[charity];
}
```

**Impact:** 
The protocol is seriously broken since it is possible to donate to an unregistered charity.

**Proof of Concept:**
The test fails. The revert never happens because the `CharityRegistry::isVerified()` call in the function `charityContract.donate()` passed.

```javascript
function testCannotDonateToUnverifiedCharity() public {
        address unverifiedCharity = address(0x4);

        // Unverified charity registers but is not verified
        vm.prank(unverifiedCharity);
        registryContract.registerCharity(unverifiedCharity);

        // Fund the donor
        vm.deal(donor, 10 ether);

        // Donor tries to donate to unverified charity
        vm.prank(donor);
        vm.expectRevert();
        charityContract.donate{value: 1 ether}(unverifiedCharity);
    }
```
Note: it is necessary to fix the error in my previous finding to have the `testCannotDonateToUnverifiedCharity` fail.

**Recommended Mitigation:** 
Instead of `CharityRegistry::isVerified()` checking the `registeredCharities`, it should check the `verifiedCharities` as follows:

```diff
function isVerified(address charity) public view returns (bool) {
-    return registeredCharities[charity];
+    return verifiedCharities[charity];
}
```
