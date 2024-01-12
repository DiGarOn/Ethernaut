![](https://ethernaut.openzeppelin.com/imgs/BigLevel21.svg)

Сan you get the item from the shop for less than the price asked?

##### Things that might help:

- `Shop` expects to be used from a `Buyer`
- Understanding restrictions of view functions

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

Solution:
