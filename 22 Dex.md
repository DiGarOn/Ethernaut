![](https://ethernaut.openzeppelin.com/imgs/BigLevel22.svg)

The goal of this level is for you to hack the basic [DEX](https://en.wikipedia.org/wiki/Decentralized_exchange) contract below and steal the funds by price manipulation.

You will start with 10 tokens of `token1` and 10 of `token2`. The DEX contract starts with 100 of each token.

You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.

### Quick note

Normally, when you make a swap with an ERC20 token, you have to `approve` the contract to spend your tokens for you. To keep with the syntax of the game, we've just added the `approve` method to the contract itself. So feel free to use `contract.approve(contract.address, <uint amount>)` instead of calling the tokens directly, and it will automatically approve spending the two tokens by the desired amount. Feel free to ignore the `SwappableToken` contract otherwise.

  Things that might help:

- How is the price of the token calculated?
- How does the `swap` method work?
- How do you `approve` a transaction of an ERC20?
- Theres more than one way to interact with a contract!
- Remix might help
- What does "At Address" do?

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
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

We will go through each function one by one.

```sol
function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
}
```

The `setTokens()` is used to set the address for each token contract. This can only be called by the owner due to the modifier `onlyOwner`.

---

```sol
function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
}
```

The `addLiquidity()` function can also be called by only the owner to provide liquidity to the contract. This transfers the approved amount of tokens from the token address to the Dex.

---

```sol
function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
}
```

- The `swap()` is a public function without any modifier which means anyone can call it. This is being used to swap `x` amount of token1 with token2 or vice-versa.
- This is taking `from` and `to` addresses for tokens and an `amount`.
- It is making sure that the addresses are only the token addresses defined by the owner using the `setTokens()` function.
- The other require statement is checking if the user calling the function has a sufficient amount of tokens.
- The variable `swapAmount` is calling the `getSwapPrice()` function to calculate the total amount to be swapped. We'll discuss more on this later.
- A `transferFrom()` call is made which is transferring `swapAmount` tokens from the user to the Dex.
- An `approve` call is being made in which the tokens to be swapped with are being approved for the contract.
- Then these `to` tokens are transferred from the Dex to our user.

---

```sol
function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
}
```

This function is taking addresses for both the tokens and the amount of `from` tokens to be swapped and calculates the amount of `to` tokens. The following formula is used -

```sol
The number of token2 to be returned = (amount of token1 to be swapped * token2 balance of the contract)/token1 balance of the contract.
```

This is the vulnerable function. We will be exploiting the fact that there are no floating points in solidity which means whenever the function will do a division, the result will be a fraction. Since there are no decimals and floating points, the token amount will be rounded off towards zero. Therefore, by making continuous token swaps from token1 to token2 and back, we can decrease the total balance of one of the tokens in the contract to zero. The precision loss will automatically do the job for us.

---

```sol
function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
}
```

The approve is an ERC20 function that is used to give permission to the spender to spend `amount` tokens.

The `balanceOf()` function is just used to calculate the remaining token balance of the address.

---
## Calculations

To exploit this level, we have to swap all our token1 for token2. Then swap all our token2 for token1. And repeat this process. Let's take a look at the token table.

1. Initially Dex has a balance of 100 for both the tokens and the User has a balance of 10 each.
2. The user makes a token swap from token1 to token2 for 10 tokens. Dex will have 110 token1 and 90 token2 whereas the user will have 0 token1 and 20 token2.
3. Now, when the user swaps 20 token2 for token1, the formula will return the following -  
```
    Number of token1 tokens returned = (20 * 110)/90 = 24.44
```
	This value will be rounded off to 24. This means Dex will now have 86 token1, and 110 token2 and our user will have 24 token1 and 0 token2. If this is repeated a few more times, it will produce the values shown below.
4. We can see that on each token swap, we are left with more tokens than held previously.  
5. Once we reach a value of 65 tokens for either token1 or token2, we can do another swap to drain the balance of one of the tokens from Dex. `((65*110)/45 = 158)`

|**Dex**||**User**||
|---|---|---|---|
|**token1**|**token2**|**token1**|**token2**|
|100|100|10|10|
|110|90|0|20|
|86|110|24|0|
|110|80|0|30|
|69|110|41|0|
|110|45|0|65|
|0|90|110|20|

This means that in the final step if we need to drain 110 token1, the amount of token2 to be swapped is `(65 * 110)/158 = 45`. This will bring the token1 balance of the Dex to 0.

---
## The Exploit

Let's take a look at our exploit script:

