### [L-03] Funders list and code not found

**Description:**

The contract docs refer to funders as " Former Participants which left their funds in the contract as donation". there is code that represents these funders. No list that records them once the deadline is over even though they are former participants.

```javascript
    ////////////////////////////////////////////////////////////////
    //////////////////       State Variables       /////////////////
    ////////////////////////////////////////////////////////////////
    address public host;
    uint256 public deadline;
    bool public deadlineSet = false;
    bool private locked = false;
    mapping(address user => bool) participant;
    mapping(address user => mapping(address token => uint256 balance)) balances;
    mapping(address user => uint256 amount) etherBalance;

    //q possible to change bool statement of ERC20. Can anyone do it?
    mapping(address token => bool) whitelisted;
```



**Impact:**

Likelyhood - Low. Being a funder does not impact the contract at all. Such state is just a social status.

Impact - Low, since funders cannot attend future events.

**Recommended Mitigation:**

```diff
////////////////////////////////////////////////////////////////
    //////////////////       State Variables       /////////////////
    ////////////////////////////////////////////////////////////////
    address public host;
    uint256 public deadline;
    bool public deadlineSet = false;
    bool private locked = false;
    mapping(address user => bool) participant;
+   mapping(address user => bool) funder;
    mapping(address user => mapping(address token => uint256 balance)) balances;
    mapping(address user => uint256 amount) etherBalance;

    //q possible to change bool statement of ERC20. Can anyone do it?
    mapping(address token => bool) whitelisted;
```

add logic to update `funder` map once once the deadline is over. Also update participant logic to reflect the end of deadline.