![](https://ethernaut.openzeppelin.com/imgs/BigLevel18.svg)

To solve this level, you only need to provide the Ethernaut with a `Solver`, a contract that responds to `whatIsTheMeaningOfLife()` with the right number.

Easy right? Well... there's a catch.

The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 opcodes at most.

Hint: Perhaps its time to leave the comfort of the Solidity compiler momentarily, and build this one by hand O_o. That's right: Raw EVM bytecode.

Good luck!

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

Solution:
## Analysis

To solve this, there's a size restriction of 10 opcodes, i.e., 10 bytes since each opcode is 1 byte. Therefore, our solver should be of at most 10 bytes and it should return 42 (0x2a).

We need to write two sets of bytecodes:

1. Initialization bytecode: It is responsible for preparing the contract and returning the runtime bytecode.
2. Runtime bytecode: This is the actual code run after the contract creation. In other words, this contains the logic of the contract.

Let's look at the Runtime opcodes first. We are using Ethereum [Docs](https://ethereum.org/en/developers/docs/evm/opcodes) for opcode reference.

---

### Runtime Opcodes

We need to do the following steps to create our runtime opcode:

1. Push and store our value (0x2a) in the memory
    
    To store the value, we'll use MSTORE(p, v) where `p` is the position or offset and `v` is the value. Since MSTORE expects the value to be already stored in the memory, we need to push it first using the PUSH1(value) opcode. We have to push both the value and the position where it'll be stored in the memory, therefore, we'll need 2 PUSH1 opcodes.
    
```sol
     1. 0x60 - PUSH1 --> PUSH(0x2a) --> 0x602a (Pushing 2a or 42)
     2. 0x60 - PUSH1 --> PUSH(0x80) --> 0x6080 (Pushing an arbitrary selected memory slot 80)
     3. 0x52 - MSTORE --> MSTORE --> 0x52 (Store value p=0x2a at position v=0x80 in memory)
```
    
2) Return the stored value
    Once we are done with the PUSH and MSTORE, it's time to return the value using RETURN(p, s) where `p` is the offset or position of our data stored in the memory and `s` is the length/size of our stored data. Therefore, we'll again need 2 PUSH1 opcodes.

```sol
     1. 0x60 - PUSH1 --> PUSH(0x20) --> 0x6020 (Size of value is 32 bytes)
     2. 0x60 - PUSH1 --> PUSH(0x80) --> 0x6080 (Value was stored in slot 0x80)
     3. 0xf3 - RETURN --> RETURN --> 0xf3 (Return value at p=0x80 slot and of size s=0x20)
```
    

We can obtain the value of bytecodes from the Docs mentioned above. Our final runtime opcode will be: `602a60805260206080f3`.

---
### Initialization Opcode

Let's take a look at the initialization opcode needed. These will be responsible for loading our runtime opcodes in memory and returning it to the EVM.

To copy code, we need to use the CODECOPY(t, f, s) opcode which takes 3 arguments.

- `t`: The destination offset where the code will be in memory. Let's save this to 0x00 offset.
- `f`: This is the current position of the runtime opcode which is not known as of now.
- `s`: This is the size of the runtime code in bytes, i.e., `602a60805260206080f3` - 10 bytes long.

```sol
1. 0x60 - PUSH1 --> PUSH(0x0a) --> 0x600a (`s=0x0a` or 10 bytes)
2. 0x60 - PUSH1 --> PUSH(0x??) --> 0x60?? (`f` - This is not known yet)
3. 0x60 - PUSH1 --> PUSH(0x00) --> 0x6000 (`t=0x00` - arbitrary chosen memory location)
4. 0x39 - CODECOPY --> CODECOPY --> 0x39 (Calling the CODECOPY with all the arguments)
```

Now, to return the runtime opcode to the EVM:

```sol
1. 0x60 - PUSH1 --> PUSH(0x0a) --> 0x600a (Size of opcode is 10 bytes)
2. 0x60 - PUSH1 --> PUSH(0x00) --> 0x6000 (Value was stored in slot 0x00)
3. 0xf3 - RETURN --> RETURN --> 0xf3 (Return value at p=0x00 slot and of size s=0x0a)
```

The bytecode for the Initialization opcode will become `600a60__600039600a6000f3` which is 12 bytes in total. This means the missing value for the starting position for the runtime opcode `f` will be index 12 or 0x0c, making our final bytecode `600a600c600039600a6000f3`.

Once we have both the bytecodes, we can combine them to get the final bytecode which can be used to deploy the contract. `602a60805260206080f3` + `600a600c600039600a6000f3` = `600a600c600039600a6000f3602a60505260206050f3`

---

## The Exploit

Here's what our exploit script looks like:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IMagicNum {
  function setSolver(address _solver) external;
}

contract PWN {
  IMagicNum instance;
  constructor (address _instance) {
    instance = IMagicNum(_instance);
  }

  function pwn() external {
    bytes memory code = "\x60\x0a\x60\x0c\x60\x00\x39\x60\x0a\x60\x00\xf3\x60\x2a\x60\x80\x52\x60\x20\x60\x80\xf3";
    address solver;
    assembly {
      solver := create(0, add(code, 0x20), mload(code))
    }
    instance.setSolver(solver);
  }
}
```

We are storing the bytecode generated above in a `code` parameter. Using assembly, we are creating a `solver` contract. The `create` opcode takes 3 inputs - value, offset, and length and returns an address of the deployed contract which is then passed into the `setSolver()` function of the Ethernaut's instance.

___
Congratulations! If you solved this level, consider yourself a Master of the Universe.

Go ahead and pierce a random object in the room with your Magnum look. Now, try to move it from afar; Your telekinesis habilities might have just started working.