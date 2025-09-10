# Reentrancy Attack in Solidity

---

## 1. What is a Reentrancy Attack?

A **Reentrancy Attack** happens when a malicious contract repeatedly calls back into a vulnerable contract **before the first function call is finished**.
This allows the attacker to drain funds or manipulate contract state unexpectedly.

The classic case is when a contract:

1. Updates balances **after** sending Ether.
2. Allows external calls during execution.

---

## 2. Types of Reentrancy Attacks

### a) **Single-Function Reentrancy**

* Attacker repeatedly calls the same vulnerable function before state variables are updated.

**Vulnerable Contract Example:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract VulnerableBank {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint256 _amount) external {
        require(balances[msg.sender] >= _amount, "Insufficient balance");

        // ❌ Vulnerability: Ether sent before balance update
        (bool sent,) = msg.sender.call{value: _amount}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] -= _amount;
    }
}
```

**Attack Contract:**

```solidity
contract Attack {
    VulnerableBank public bank;

    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }

    fallback() external payable {
        if (address(bank).balance > 1 ether) {
            bank.withdraw(1 ether); // Re-enters before balance update
        }
    }

    function attack() external payable {
        bank.deposit{value: 1 ether}();
        bank.withdraw(1 ether);
    }
}
```

---

### b) **Cross-Function Reentrancy**

* The attacker uses **two different functions** to exploit the vulnerability.

**Example:**

```solidity
contract TimeLock {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint bal = balances[msg.sender];
        require(bal > 0, "No balance");

        (bool sent,) = msg.sender.call{value: bal}("");
        require(sent, "Failed");

        balances[msg.sender] = 0; // ✅ but happens after send
    }

    function transfer(address _to, uint _amount) public {
        require(balances[msg.sender] >= _amount, "Not enough balance");
        balances[msg.sender] -= _amount;
        balances[_to] += _amount;
    }
}
```

**Attack Flow:**

1. Call `withdraw()` → reenter.
2. During fallback, call `transfer()` to move balances around before withdrawal finishes.

---

### c) **Cross-Contract Reentrancy**

* The vulnerable contract calls another contract, which then calls back into the first one **before state is updated**.

**Example:**

```solidity
contract Victim {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function pay(address _to, uint _amount) external {
        require(balances[msg.sender] >= _amount, "Not enough");

        (bool sent,) = _to.call{value: _amount}(""); 
        require(sent, "Failed");

        balances[msg.sender] -= _amount;
    }
}
```

Here, if `_to` is a malicious contract, it can reenter `pay()` before the sender’s balance is reduced.

---

### d) **Read-Only Reentrancy**

* The attacker **reenters during a `view` or `pure` function call** that relies on state which can be manipulated mid-execution.
* Example: Oracles or price feeds calling into contracts that depend on temporarily inconsistent state.

**Example (conceptual):**

```solidity
contract PriceOracle {
    uint public price = 100;

    function updatePrice(uint _newPrice) external {
        // Imagine this triggers external callback
        price = _newPrice;
    }

    function getPrice() external view returns (uint) {
        return price;
    }
}
```

An attacker could reenter during the update process if the oracle makes an external call.

---

## 3. Preventing Reentrancy

1. **Checks-Effects-Interactions Pattern**

   * Always update state before making external calls.

   ```solidity
   function withdraw(uint _amount) external {
       require(balances[msg.sender] >= _amount, "Insufficient");
       balances[msg.sender] -= _amount; // ✅ effect first

       (bool sent,) = msg.sender.call{value: _amount}("");
       require(sent, "Failed");
   }
   ```

2. **Reentrancy Guard (Mutex)**

   * Use a guard variable or OpenZeppelin’s `ReentrancyGuard`.

   ```solidity
   import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

   contract SecureBank is ReentrancyGuard {
       mapping(address => uint) public balances;

       function withdraw(uint _amount) external nonReentrant {
           require(balances[msg.sender] >= _amount, "Insufficient");
           balances[msg.sender] -= _amount;
           (bool sent,) = msg.sender.call{value: _amount}("");
           require(sent, "Failed");
       }
   }
   ```

3. **Avoid `call` for Sending Ether**

   * Use `transfer` or `send` (they forward only 2300 gas).
   * But be aware: since EIP-1884, gas costs have changed, so many prefer `call` with reentrancy guards.

---

## 4. Summary

* **Single-function reentrancy** → attacker repeatedly calls the same function.
* **Cross-function reentrancy** → attacker switches between functions.
* **Cross-contract reentrancy** → attacker exploits multiple contracts.
* **Read-only reentrancy** → attacker manipulates state during read calls.

✅ Best practice: **Use checks-effects-interactions and reentrancy guards**.

---
