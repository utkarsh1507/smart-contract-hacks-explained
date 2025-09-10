
---

# Weak Randomness Attack

## What is it?

Smart contracts sometimes need randomness (e.g., lotteries, gambling, NFT traits).
However, **Ethereum is a deterministic system** — every node must reach the same result, so randomness is not natively available.
Developers often try to generate “random” numbers using on-chain values like:

* `block.timestamp`
* `block.difficulty` (before EIP-1559)
* `block.number`
* `msg.sender`

⚠️ These are **not truly random** and can be influenced or predicted by miners/validators or other participants.

---

## Example Vulnerable Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract WeakRandomness {
    uint256 public winningNumber;

    function pickWinner() public {
        // ❌ Weak randomness
        winningNumber = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender)
            )
        ) % 10; // Random number between 0–9
    }
}
```

### Why is this weak?

1. **Predictable**: Anyone can compute the hash since `block.timestamp`, `difficulty`, and `msg.sender` are public.
2. **Miner manipulation**: A miner can adjust the block timestamp slightly to influence the result.
3. **Front-running**: Attackers can see the transaction in the mempool and decide to call or skip based on whether they’d win.

---

## Attack Scenario

Imagine this contract runs a **lottery** where if `winningNumber == 7`, the caller wins the prize.

* An attacker simulates `pickWinner()` **off-chain** (using known block values and their own address).
* If they see they won → they send the transaction.
* If they didn’t → they don’t send (or manipulate gas to delay).
* In mining/validation, miners can even adjust the timestamp to help themselves win.

This allows attackers to bias the outcome heavily.

---

## Safer Alternatives

### 1. **Commit-Reveal Scheme**

* Users commit to a random seed (hash of a secret).
* Later they reveal the secret.
* The contract verifies and derives randomness.
* Prevents front-running because the attacker can’t know the secret in advance.

### 2. **Chainlink VRF (Verifiable Random Function)**

* A decentralized oracle that provides **provably fair randomness**.
* Randomness is cryptographically verified on-chain.

```solidity
// Example only (simplified)
import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract SecureRandomness is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;

    constructor(address vrfCoordinator, address linkToken, bytes32 _keyHash, uint256 _fee) 
        VRFConsumerBase(vrfCoordinator, linkToken)
    {
        keyHash = _keyHash;
        fee = _fee;
    }

    function getRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK");
        return requestRandomness(keyHash, fee);
    }

    function fulfillRandomness(bytes32, uint256 randomness) internal override {
        randomResult = randomness;
    }
}
```

### 3. **Off-chain randomness with multiple revealers**

* Multiple parties submit randomness.
* Final random number is XOR of all.
* No single party controls the outcome.

---

## Best Practices

* Never rely on **block variables** for randomness in high-value apps.
* Use **commit-reveal** or **Chainlink VRF** for fairness.
* Always assume miners and users will act adversarially.

---

