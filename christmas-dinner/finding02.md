### [M-02] It is not possible to sign up a friend

**Description:**

In `ChristmasDinner::deposit()` there is no way to identify if the `message.sender` is signing up a friend or himself.

```javascript
    function deposit(address _token, uint256 _amount) external beforeDeadline {
        //q should sign up have a minimum amount?
        if (!whitelisted[_token]) {
            revert NotSupportedToken();
        }
        if (participant[msg.sender]) {
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
            participant[msg.sender] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit NewSignup(
                msg.sender,
                _amount,
                getParticipationStatus(msg.sender)
            );
        }
    }
```
`msg.sender` reflects who is calling the contract. Therefore, only his address will show up on the `participant` list.


**Impact:**

Likelyhood - High, it is normal for someone to sign up friends for events.

Impact - Medium/low, no funds are lost, but the contract does not allow to sign up friends as promised.


**Proof of Concept:**

```javascript
    function test_signUpFriends() public {
        vm.startPrank(user1);
        cd.deposit(address(wbtc), 1e18);
        vm.stopPrank();
        assertEq(cd.getParticipationStatus(user2), true);
    }
```
`user1` wants to sign up his friend, `user2`. The problem appears when there is no option in the contract to do that. Let's look at `ChristmasDinner::deposit()`:

```javascript
1-> function deposit(address _token, uint256 _amount) external beforeDeadline {
        if (!whitelisted[_token]) {
            revert NotSupportedToken();
        }
        if (participant[msg.sender]) {
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
 2->        participant[msg.sender] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit NewSignup(
                msg.sender,
                _amount,
                getParticipationStatus(msg.sender)
            );
        }
    }
```
There should be an option to send the address we want to sign up. If `user1` wants to sign up `user2`, `user1` should send the address of his friend. The deposit function does not allow that because:
1. has no parameter to receive an address,
2. only changes the `participant` list with `msg.sender`.


**Recommended Mitigation:**

```diff
-   function deposit(address _token, uint256 _amount) external beforeDeadline {
+   function deposit(address _token, uint256 _amount, address _signupAddress) external beforeDeadline {
        //q should sign up have a minimum amount?
        if (!whitelisted[_token]) {
            revert NotSupportedToken();
        }
-        if (participant[msg.sender]) {
+        if (participant[_signupAddress]) {
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
-           participant[msg.sender] = true;
+           participant[_signupAddress] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit NewSignup(
                msg.sender,
                _amount,
                getParticipationStatus(msg.sender)
            );
        }
    }
```

Add a function parameter with the address and update the function deposit accordingly. `_signupAddress` can be `msg.sender`.