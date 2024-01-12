![](https://ethernaut.openzeppelin.com/imgs/BigLevel12.svg)

task is to unlock contract 


```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```


Solution:
`player` has to set `locked` state variable to `false`.

This is similar to level 8: Vault where any state variable (irrespective of whether it is `private`) can be read given it's slot number.

`unlock` uses the third entry (index 2) of `data` which is a `bytes32` array. Let's determined `data`'s third entry's slot number (each slot can accommodate at most 32 bytes) according to storage rules:

- `locked` is 1 byte `bool` in slot 0
- `ID` is a 32 byte `uint256`. It is 1 byte extra big to be inserted in slot 0. So it goes in & totally fills slot 1
- `flattening` - a 1 byte `uint8`, `denomination` - a 1 byte `uint8` and `awkwardness` - a 2 byte `uint16` totals 4 bytes. So, all three of these go into slot 2
- Array data always start a new slot, so `data` starts from slot 3. Since it is `bytes32` array each value takes 32 bytes. Hence value at index 0 is stored in slot 3, index 1 is stored in slot 4 and index 2 value goes into slot 5

Alright so key is in slot 5 (index 2 / third entry). Read it.  

```js
key = await web3.eth.getStorageAt(contract.address, 5)

// Output: '0x5dd89f7b81030395311dd63330c747fe293140d92dbe7eee1df2a8c233ef8d6d'
```

This `key` is 32 byte. But `require` check in `unlock` converts the `data[2]` 32 byte value to a `byte16` before matching.

`byte16(data[2])` will truncate the last 16 bytes of `data[2]` and return only the first 16 bytes.

Accordingly convert `key` to a 16 byte hex (with prefix - `0x`):  

```js
key = key.slice(0, 34)

// Output: 0x5dd89f7b81030395311dd63330c747fe
```

Call `unlock` with `key`:  

```js
await contract.unlock(key)
```

Unlocked! Verify by:  

```js
await contract.locked()

// Output: false
```
___
Nothing in the ethereum blockchain is private. The keyword private is merely an artificial construct of the Solidity language. Web3's `getStorageAt(...)` can be used to read anything from storage. It can be tricky to read what you want though, since several optimization rules and techniques are used to compact the storage as much as possible.

It can't get much more complicated than what was exposed in this level. For more, check out this excellent article by "Darius": [How to read Ethereum contract storage](https://medium.com/aigang-network/how-to-read-ethereum-contract-storage-44252c8af925)