### [H-01] `GivingThanks` constructor incorrectly assigns the address of `CharityRegistry` making it impossible to donate to charities

**Description:** 
The variable `registry` is storing the address of the caller `msg.sender`and not the address of the charity stored.

```javascript
    constructor(address _registry) ERC721("DonationReceipt", "DRC") {
@>      registry = CharityRegistry(msg.sender);
        owner = msg.sender;
        tokenCounter = 0;
    }
```

 Subsequent calls to `GivingThanks::donate()` will not be allowed. It fails the required condition. The address stored will not be `CharityRegistry` and therefore will not pass the `CharityRegistry::isVerified()` 

```javascript
    function donate(address charity) public payable {
@>      require(registry.isVerified(charity), "Charity not verified");
        (bool sent, ) = charity.call{value: msg.value}("");
        require(sent, "Failed to send Ether");

        _mint(msg.sender, tokenCounter);

        // Create metadata for the tokenURI
        string memory uri = _createTokenURI(
            msg.sender,
            block.timestamp,
            msg.value
        );
        _setTokenURI(tokenCounter, uri);

        tokenCounter += 1;
    }
```

**Impact:** 
Donation is a core functionality of the protocol. The protocol is seriously disrupted if no donations are allowed.

**Proof of Concept:**

Unit test `testDonate()`

```javascript
function testDonate() public {
        uint256 donationAmount = 1 ether;

        // Check initial token counter
        uint256 initialTokenCounter = charityContract.tokenCounter();

        // Fund the donor
        vm.deal(donor, 10 ether);

        // Donor donates to the charity
        vm.prank(donor);
        charityContract.donate{value: donationAmount}(charity);

        // Check that the NFT was minted
        uint256 newTokenCounter = charityContract.tokenCounter();
        assertEq(newTokenCounter, initialTokenCounter + 1);

        // Verify ownership of the NFT
        address ownerOfToken = charityContract.ownerOf(initialTokenCounter);
        assertEq(ownerOfToken, donor);

        // Verify that the donation was sent to the charity
        uint256 charityBalance = charity.balance;
        assertEq(charityBalance, donationAmount);
    }
```
Test testDonate() failed as copied from terminal:

Ran 1 test for test/GivingThanks.t.sol:GivingThanksTest
[FAIL. Reason: EvmError: Revert] testDonate() (gas: 27751)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 23.69ms (3.80ms CPU time)

Ran 1 test suite in 36.11ms (23.69ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/GivingThanks.t.sol:GivingThanksTest
[FAIL. Reason: EvmError: Revert] testDonate() (gas: 27751)

Encountered a total of 1 failing tests, 0 tests succeeded


**Recommended Mitigation:**
Correctly store the address of the charity in `registry`:

```diff
    constructor(address _registry) ERC721("DonationReceipt", "DRC") {
-        registry = CharityRegistry(msg.sender);
+        registry = CharityRegistry(_registry);
        owner = msg.sender;
        tokenCounter = 0;
    }
```
