# Denial of Service (DoS) Attack in Solidity

---

## 1. What is a DoS Attack?

A **Denial of Service (DoS) attack** in Solidity happens when a malicious actor prevents other users from interacting with a contract or blocks the contract from executing normally.
Unlike reentrancy, DoS attacks focus on **disrupting availability** rather than directly stealing funds.

---

## 2. Types of DoS Attacks

### a) DoS with Unexpected Revert

If a contract calls another contract without handling possible reverts, a malicious contract can force the entire function to fail.

**Example:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract Auction {
    address public highestBidder;
    uint public highestBid;

    function bid() external payable {
        require(msg.value > highestBid, "Bid too low");

        // Refund old bidder
        if (highestBidder != address(0)) {
            (bool sent,) = highestBidder.call{value: highestBid}("");
            require(sent, "Refund failed"); // ❌ vulnerable
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }
}
```

**Attack Flow:**

1. Malicious user bids once.
2. Their fallback function always reverts.
3. When someone else tries to outbid, refund fails → `require(sent)` fails → new bid rejected.
4. Attacker becomes **permanent highest bidder**.

---

### b) DoS with Block Gas Limit

When looping over a dynamic array or mapping, if the loop grows too large, execution may exceed the block gas limit, causing a **permanent DoS**.

**Example:**

```solidity
contract Airdrop {
    address[] public recipients;

    function addRecipient(address _user) external {
        recipients.push(_user);
    }

    function sendTokens() external {
        for (uint i = 0; i < recipients.length; i++) {
            // If recipients list is huge, this may run out of gas ❌
            payable(recipients[i]).transfer(1 ether);
        }
    }
}
```

**Attack Flow:**

1. Attacker adds thousands of addresses.
2. `sendTokens()` exceeds block gas limit.
3. Airdrop becomes unusable forever.

---

### c) DoS with Unexpected Self-Destruct or Forced Ether

Contracts relying on `balance` checks or assumptions about state can be broken if an attacker uses `selfdestruct` to send Ether forcibly.

**Example:**

```solidity
contract Fund {
    function withdraw() external {
        require(address(this).balance == 0, "Balance not empty");
        // Logic depending on zero balance
    }
}
```

**Attack:**

* A malicious contract can use `selfdestruct` to force Ether into this contract.
* `withdraw()` will always fail since balance is never zero again.

---

### d) DoS by Block Timestamp Manipulation

If critical logic depends on `block.timestamp`, miners can manipulate it slightly to their advantage.

**Example:**

```solidity
contract Lottery {
    uint public endTime;
    address public winner;

    constructor(uint duration) {
        endTime = block.timestamp + duration;
    }

    function pickWinner() external {
        require(block.timestamp >= endTime, "Too early");
        winner = msg.sender; // Vulnerable to miner manipulation
    }
}
```

Attackers (especially miners) can adjust `block.timestamp` slightly to make sure they win.

---

## 3. Preventing DoS Attacks

1. **Pull over Push pattern**

   * Let users withdraw their own funds instead of pushing Ether automatically.

   ```solidity
   mapping(address => uint) public refunds;

   function bid() external payable {
       require(msg.value > highestBid, "Low bid");

       if (highestBidder != address(0)) {
           refunds[highestBidder] += highestBid; // Store refund instead of sending
       }

       highestBidder = msg.sender;
       highestBid = msg.value;
   }

   function withdrawRefund() external {
       uint amount = refunds[msg.sender];
       refunds[msg.sender] = 0;
       payable(msg.sender).transfer(amount);
   }
   ```

2. **Avoid unbounded loops**

   * Use pagination or process in batches.

3. **Do not rely on contract balance being zero**

   * Track balances with state variables instead.

4. **Avoid critical dependence on `block.timestamp`**

   * Use block numbers and add tolerance.

---

## 4. Summary

* **DoS with revert** → attacker prevents others by reverting on refunds.
* **DoS with gas limit** → contract breaks due to unbounded loops.
* **DoS with forced Ether** → attacker disrupts balance assumptions.
* **DoS with timestamp manipulation** → attacker or miner exploits block time.

✅ Best practice: prefer **pull over push**, avoid unbounded loops, and carefully design logic to handle failures.
