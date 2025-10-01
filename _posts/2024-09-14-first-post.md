---
layout: post
title: "Getting Started with Smart Contract Security"
date: 2024-09-14
categories: [web dev, security, learning]
---

# Getting Started with Smart Contract Security

This is my journey into learning smart contract security and auditing.

## What I've learned so far:

- Basic Solidity vulnerabilities
- How to use security tools like Slither and Mythril
- Common attack vectors in DeFi protocols

## Key takeaways:

```solidity
// Example of a vulnerable contract
contract Vulnerable {
    mapping(address => uint) public balances;
    
    function withdraw() public {
        uint amount = balances[msg.sender];
        require(amount > 0);
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        balances[msg.sender] = 0;
    }
}