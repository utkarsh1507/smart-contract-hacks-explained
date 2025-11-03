# Arithmetic Overflow / Underflow Attack in Solidity

---

## 1. What is Arithmetic Overflow / Underflow in Solidity?

* In Solidity versions **<0.8.0**, arithmetic operations on `uint` (unsigned integers) **do not revert** when they exceed their maximum or minimum values.
* Instead, they **wrap around**:

  * **Overflow**: if result > `2^256 - 1`, it wraps back to `0`.

    ```solidity
    uint8 x = 255;     
    x = x + 1; // x becomes 0    
    ```
  * **Underflow**: if result < `0`, it wraps around to the maximum value.

    ```solidity
    uint8 y = 0; 
    y = y - 1; // y becomes 255
    ```

This behavior can be exploited if contracts rely on arithmetic for security conditions.

---

## 2. Understanding the Vulnerable Contract (`TimeLock`)

The **TimeLock** contract is meant to:

* Allow deposits.
* Force users to wait at least **1 week** before withdrawing.
* Let users extend their lock time.

### Functions:

* `deposit()`:

  * Stores deposited Ether in `balances[msg.sender]`.
  * Sets `lockTime[msg.sender] = block.timestamp + 1 weeks`.

* `increaseLockTime(uint256 _secondsToIncrease)`:

  * Increases the lock time.
  * **Vulnerability**: If `lockTime` overflows, it can wrap to a very small value or even `0`.

* `withdraw()`:

  * Requires:

    * The user has funds.
    * `block.timestamp > lockTime[msg.sender]`.
  * Transfers funds.

---

## 3. The Attack

The `Attack` contract exploits the overflow.

### Attack Strategy:

1. Deposit Ether (so lockTime is set to `now + 1 week`).
2. Calculate how much to **increase** lock time so it overflows back to `0`:

   ```solidity
   x = type(uint256).max + 1 - lockTime[address(this)]
   ```

   * Here `type(uint256).max = 2^256 - 1`.
   * Adding this value causes:

     ```
     lockTime + x = 2^256 = 0 (wrapped around)
     ```
3. After overflow, `lockTime[address(this)]` becomes `0`.
4. Since `block.timestamp > 0`, withdrawal succeeds immediately.

---

## 4. Walkthrough of the Given Code

### TimeLock

```solidity
function deposit() external payable {
    balances[msg.sender] += msg.value; // Track user funds
    lockTime[msg.sender] = block.timestamp + 1 weeks; // Enforce 1 week wait
}
```

```solidity
function increaseLockTime(uint256 _secondsToIncrease) public {
    lockTime[msg.sender] += _secondsToIncrease; // Vulnerable: no overflow check
}
```

```solidity
function withdraw() public {
    require(balances[msg.sender] > 0, "Insufficient funds");
    require(block.timestamp > lockTime[msg.sender], "Lock time not expired");

    uint256 amount = balances[msg.sender];
    balances[msg.sender] = 0;

    (bool sent,) = msg.sender.call{value: amount}("");
    require(sent, "Failed to send Ether");
}
```

### Attack

```solidity
function attack() public payable {
    timeLock.deposit{value: msg.value}();

    // Craft malicious lock time increment
    timeLock.increaseLockTime(
        type(uint256).max + 1 - timeLock.lockTime(address(this))
    );

    // Withdraw immediately (lockTime = 0 after overflow)
    timeLock.withdraw();
}
```

---

## 5. Why This Works Only in Solidity <0.8.0

* In Solidity `0.8.0` and above, arithmetic overflow/underflow **throws an error** automatically.
* That’s why in newer contracts, this attack won’t work unless the developer explicitly disables checks with `unchecked { ... }`.

---

## 6. Fixing the Vulnerability

To fix it (for Solidity <0.8.0):

* Use **SafeMath** library (from OpenZeppelin) to handle overflows safely.

Example:

```solidity
using SafeMath for uint256;

function increaseLockTime(uint256 _secondsToIncrease) public {
    lockTime[msg.sender] = lockTime[msg.sender].add(_secondsToIncrease);
}
```

In Solidity ≥0.8.0:

* No extra library is needed since overflow checks are built-in.

---

✅ In short:
This exploit relies on **overflowing the `lockTime`** so that the waiting period is bypassed, allowing instant withdrawal.
