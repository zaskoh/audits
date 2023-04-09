## CollatzPuzzle solver

To solve the puzzle we need to deploy a contract with a size > 0 and <= 32. Also the contract need to return the right calculation for the collatz function.

As the size is pretty small we will need to deploy a yul contract that only calculate the equation and returns.

```solidity
object "YulCollatz" {
    code {
        // deploy contract
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }

    object "runtime" {
        code {
            let mem := mload(0x40) // get a free memory pointer

            // n = calldataload(4)

            if gt(mod(calldataload(4), 2), 0) { // check for n % 2 > 0
                mstore(mem, add(mul(calldataload(4), 3), 1)) // save 3 * n + 1 to memory
                return(mem, 32) // return early
            }

            mstore(mem, div(calldataload(4), 2)) // save n / 2 to memory
            return(mem, 32)
        }
    }
}
```

To get the bytecode that we need to deploy we can use solc version 0.8.17 and use the following command:
```bash
solc --strict-assembly --optimize --optimize-runs 200 --bin yul/YulCollatz.yul

======= yul/YulCollatz.yul (EVM) =======

Binary representation:
601f80600c6000396000f3fe60206040516004356001811660155760011c8152f35b6003026001018152f3
```

With this bytecode we will have a sizecode of only 31 and will pass the size check. The return value is calculated based on the equation and returned.

## Test
forge test --match-contract=CollatzPuzzleTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract CollatzPuzzleTest is Test {

    address solverContract;
    CollatzPuzzle collatzPuzzle;

    function setUp() public {
        
        // deploy our contract
        bytes memory _bytecode = hex"601f80600c6000396000f3fe60206040516004356001811660155760011c8152f35b6003026001018152f3";
        address tmpAddr;
        assembly {
            tmpAddr := create(0, add(_bytecode, 0x20), mload(_bytecode))
        }
        require(tmpAddr != address(0), "contract creation failed");
        solverContract = tmpAddr;

        // deploy puzzle
        collatzPuzzle = new CollatzPuzzle();
    }

    function testAttack() public {     
        bool success = collatzPuzzle.callMe(solverContract);
        assertTrue(success, "we should not revert !");
    }
}

interface ICollatz {
  function collatzIteration(uint256 n) external pure returns (uint256);
}

contract CollatzPuzzle is ICollatz {
  function collatzIteration(uint256 n) public pure override returns (uint256) {
    if (n % 2 == 0) {
      return n / 2;
    } else {
      return 3 * n + 1;
    }
  }

  function callMe(address addr) external view returns (bool) {
    // check code size
    uint256 size;
    assembly {
      size := extcodesize(addr)
    }
    require(size > 0 && size <= 32, "bad code size!");

    // check results to be matching
    uint p;
    uint q;
    for (uint256 n = 1; n < 200; n++) {
      // local result
      p = n;
      for (uint256 i = 0; i < 5; i++) {
        p = collatzIteration(p);
      }
      // your result
      q = n;
      for (uint256 i = 0; i < 5; i++) {
        q = ICollatz(addr).collatzIteration{gas: 100}(q);
      }
      require(p == q, "result mismatch!");
    }

    return true;
  }
}
```