![](https://ethernaut.openzeppelin.com/imgs/BigLevel7.svg)

Some contracts will simply not take your money `¯\_(ツ)_/¯`

The goal of this level is to make the balance of the contract greater than zero.

  Things that might help:

- Fallback methods
- Sometimes the best way to attack a contract is with another contract.
- See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

## Fun fact: `selfdestruct` is currently 1 of 3 methods for your contract to receive ether

**Method 1 — via payable functions:** Earlier, we discussed that the [fallback function](https://hackernoon.com/ethernaut-lvl-1-walkthrough-how-to-abuse-the-fallback-function-118057b68b56) is to _intentionally_ allow your contract to receive Ether from other contracts and external wallets. But if no such `payable` function exists, your contract still has 2 more indirect ways of receiving funds:

**Method 2 — receiving mining reward:** contract addresses can be designated as the recipients of mining block rewards.

**Method 3 — from a destroyed contract:** As discussed, `selfdestruct` lets you designate a backup address to receive the remaining ethers from the contract you are destroying.

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SelfDestructingContract {
	constructor() payable {}
	
	function selfDestroy(address payable addr) public {
		selfdestruct(addr);
	}

}
```

___
In solidity, for a contract to be able to receive ether, the fallback function must be marked `payable`.

However, there is no way to stop an attacker from sending ether to a contract by self destroying. Hence, it is important not to count on the invariant `address(this).balance == 0` for any contract logic.