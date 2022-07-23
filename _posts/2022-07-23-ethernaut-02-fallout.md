---
title: Ethernaut 02 Fallout
author: man715
date: 2022-07-23
categories: [Smart Contract, Write-up]
tags: [write-up, hacking, smart-contract]
math: true
mermaid: true
image:
  path: 
  width: 800
  height: 500
  alt: 
---

# Introduction
The objective for this challenge is it take ownership of the contract. 

To be honest, this challenge is pretty pointless in and of itself; however, I think we can still learn something from it. Let's take a look.

According to the solidity documentation,
> Prior to version 0.4.22, constructors were defined as functions with the same name as the contract. This syntax was deprecated and is not allowed anymore in version 0.5.0.
{: .prompt-warning }

Although now this contract's pragma is `^0.6.0`, it was originally compiled with a version in the `0.4.xx`.

# Assessment
If you look closely, you will notice that there is a function named `Fal1out` which is not the same as the contract name which is `Fallout`. 

## Vulnerability
The function that sets the owner can be called by anyone. The intention was for it to be the constructor function and set the creator's address as the owner. Unfortunately, this mistake just created a function that will set any address that calls it as the owner. 

# Exploitation
To exploit this contract, all we have to do is to call the `Fal1out` function. 

```javascript
 contract.Fal1out({ from: player, value: toWei(".0001", "ether") })
```

# Lesson(s)
Although this particular vulnerability has been fixed at compilation, there is still something that we can take from this challenge. Attention to detail is critical when it comes to reviewing code. Do not make assumptions on what something is doing. It could vary well be that the implementation of that portion of the code is not doing what you and the author thought at first glance. Any critical functions need to be understood fully. 


# Fuzzing with Echidna
## Setup
This contract does not require any setup.

## Test Setup
This requires a very simple test of check if the owner is the msg.sender. If it is, the test should return false since that is not the ideal result.

```javascript
function echidna_is_owner() public view returns(bool) {
        if (owner == msg.sender) {
            return false;
        } else {
            return true;
        }
```

## Run the test
```shell
echidna-test Test.sol --contract Test
```