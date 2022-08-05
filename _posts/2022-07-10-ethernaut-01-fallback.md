---
title: Ethernaut 01 Fallback
author: man715
date: 2022-07-09
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

The introduction gives us four hints so first let's check those out. 

- How to send ether when interacting with an ABI
- How to send ether outside of the ABI
- Converting to and from wei/ether units
- Fallback methods

## Fallback Function
The fallback function is designed to be called under specific circumstances.
- The function called is not found.
- The contract was sent Ether without any data.

For this contract we are only really concerned with the `receive()` function which will be called by instead of the fallback function when it exists and the contract was sent Ether without any data.


## Sending Ether to a contract
There are a few ways you can send Ether to a contract. The first is by calling a function on the target contract that does not exist. The contract will always default to using the fallback function if the function is not found. You can also trigger the fallback function by sending Ether to the contract without sending any data with the transaction. This will trigger the receive function if it exists but will trigger the fallback if the receive function does not.

## Converting Wei/Ether
Solidity uses [fixed-point arithmetic](https://en.wikipedia.org/wiki/Fixed-point_arithmetic). Fixed-point arithmetic represents fractional values as a multiple of some fixed unit size. In the case of Ether, that size is 10^18(1000000000000000000). This means that one Ether is 1*10^18 and two Ether is 2*10^18. The smallest unit of Ether is known as a Wei. So one Wei is represented as 1. The other important unit of size is the Gwei which is 10*9. This is important because that is the unit of measurement used for gas.

To convert 10 Ether to Wei: 10 \* 10^18. To convert 10000000000000000000 Wei to Ether: 10000000000000000000 / 10^18.

Frameworks such as web3.js or web3.py can convert it for using unit size name. 

The Solidity language also allows you to use the unit size name to convert as well. The example from the documentation is as follows.
```
assert(1 wei == 1);
assert(1 gwei == 1e9);
assert(1 ether == 1e18)
```

The objective of this challenge is to claim ownership of the contract and reduce the contract's balance to 0.

# Assessment
Since we already know our objective is to put our address in the owner variable, we will first look at all the places that is set. The first place is in the constructor but that has already ran during the construction of the contract which sets the variable to the address of the contract creator's. 

The next place where the owner can be set is in the `contribute()` function and can be called by anyone which can be seen by the fact it is a public function and does not have the `onlyOwner` modification. This function allows a user to send less than 0.001 ether (1000000000000000 Wei) to the contract. If the user has more contributions than the current owner, the user becomes the owner. Maybe we can get lucky and the owner has not contributed anything or just very little! 

```javascript
await contract.owner()
'0x9CB391dbcD447E645D6Cb55dE6ca23164130D008'
BigInt(await contract.contributions('0x9CB391dbcD447E645D6Cb55dE6ca23164130D008'))
1000000000000000000000n
```
Unfortunately, the owner has 1000000000000000000000 Wei or 100 Ether in contributions. That means would need to make more than 1 * 10^24 transactions! I don't think we have enough Ether to cover the gas for that many transactions let alone have 100 Ether to contribute. Let's see if we can find another way. It is interesting to note that although the contract says that the owner has 100 Ether worth of contributions, the contract's balance is 0 which can be seen with `await web3.eth.getBalance(contract.address)`.

Okay the `receive()` function also changes the owner and can be called by anyone. Let's take a closer look at this function since it is short it should be pretty easy.

```
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

As we discussed before, the receive function will be called if anyone sends the contract any Ether without any data. Since the function is payable, it will not reject the transaction. Lastly, the function is external which means that only calls to this contract from outside of itself can call this contract. Meaning that any other contract or externally owned account (user account) can make a call to this contract. The require statement is check that the transaction contains at least 1 Wei and that the sender has contributed to the contract before by calling the `contribute()` function. If we can satisfy both criteria, we can take over ownership. Notice how the check does not verify that the transaction sender has more Ether than the owner. This means that we can submit any amount to the `contribute()` function and to the `receive()` function and take over the contract. 

## Vulnerability
The fallback and receive functions should be simple functions that do not have any state changes let alone perform critical functions. These functions can be called by anyone at any time. 


# Exploitation
Exploiting this contract is fairly straight forward once you understand what it is doing. First, we need to make a contribution. Next, we need to send some Ether to the contract. 

```javascript
await contract.contribute({value: 1})
await sendTransaction({from: player, to: contract.address, value: toWei('0.000001')})
await contract.withdraw()
```

# Lesson(s)
While you should implement a fallback and receive function, you should use caution if either of those two functions perform any state changes or critical functionality.
The fallback and receive functions should be used to check conditions instead of performing critical logic.

# Fuzzing with Echidna
The files used can be found here: https://github.com/man715/echidna-ctf
Looking at the contract and the goal, we know we want to take ownership and drain the contract of all of its funds. 

## Setup
First, we should set up the config file so that the sender address is predictable and not randomized and make sure that the contract is supplied with some ether. 

config.yaml:
```yaml
sender: ["0x10000"]
balanceContract: 1000
```

Next, modify the target contract. Unfortunately, Echidna cannot reach `receive()` as a fallback function yet; however, it is being [worked on](https://github.com/crytic/echidna/pull/722). So to make sure we can reach all functions, we need to put `function` in front of the `recieve()` function definition. 

## Test Setup 
To test for this we can create a very simple test function.
```javascript
function echidna_owner_and_drain_funds() public view returns(bool) {
        if (owner == msg.sender) {
            return address(this).balance >= 1000;
        } else {
            return true;
        }
```

This function first checks to make sure the sender has owner ship of the contract then verifies if the contract still has the amount of ether we initially supplied it with.

## Run Test
Now run echidna from within the docker image.
```bash
echidna-test --config config.yaml Test.sol --contract Test
```

https://github.com/man715/echidna-ctf/tree/main/ethernaut