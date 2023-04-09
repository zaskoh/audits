## Gate solver

To solve the puzzle we need a contract with a size <= 32. Also the contract need to return on the function `f00000000_bvvvdlt` (0x00000000) the caller and on the function `f00000001_grffjzz` (0x00000001) the origin caller. The third function `fail()` needs to revert when called.

As the size is pretty small we will need an optimized byte-code contract. The following opcodes will handle all the conditions and will be in the required size.

```bash
PUSH1 0x00
DUP1
CALLDATALOAD
DUP1
PUSH1 0xf8
SHR
PUSH1 0x1c
JUMPI
SWAP1
PUSH1 0x20
SWAP2
ORIGIN
DUP3
MSTORE
ISZERO
PUSH1 0x17
JUMPI
RETURN
JUMPDEST
CALLER
DUP2
MSTORE
RETURN
JUMPDEST
POP
DUP1
REVERT
```

This translates to the following run-code:
`600080358060f81c601c579060209132825215601757f35b338152f35b5080fd`

The deployed byte-code will be:
`602080600c6000396000f3fe600080358060f81c601c579060209132825215601757f35b338152f35b5080fd`

This is the yul-contract to get the byte-code
```solidity
object "YulGate" {
    code {
        // deploy contract
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }

    object "runtime" {
        code {
            if gt(shr(0xf8, calldataload(0)), 0) {
                revert(0, 0)
            }

            mstore(0, origin())

            if iszero(calldataload(0)) {
                mstore(0, caller())
            }

            return(0, 32)
        }
    }
}
```

To verify the bytecode that we need to deploy we can use solc version 0.8.17 and use the following command:
```bash
solc --strict-assembly --optimize --optimize-runs 200 --bin yul/YulGate.yul

======= yul/YulGate.yul (EVM) =======

Binary representation:
602080600c6000396000f3fe600080358060f81c601c579060209132825215601757f35b338152f35b5080fd
```

With this bytecode we will have a sizecode of exactly 32 and will pass all the other checks.

## Test
forge test --match-contract=GateTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract GateTest is Test {

    address bob = makeAddr("bob");
    address guardianContract;
    Gate gate;

    function setUp() public {
        
        // deploy our contract
        bytes memory _bytecode = hex"602080600c6000396000f3fe600080358060f81c601c579060209132825215601757f35b338152f35b5080fd";
        address tmpAddr;
        assembly {
            tmpAddr := create(0, add(_bytecode, 0x20), mload(_bytecode))
        }
        require(tmpAddr != address(0), "contract creation failed");
        guardianContract = tmpAddr;

        // deploy gate
        gate = new Gate();
    }

    function testOpenTheGate() public {
        assertFalse(gate.opened(), "gate should be closed!");
        
        vm.prank(bob, bob);        
        gate.open(guardianContract);

        assertTrue(gate.opened(), "gate should be open!");
    }
}

interface IGuardian {
    function f00000000_bvvvdlt() external view returns (address);

    function f00000001_grffjzz() external view returns (address);
}

contract Gate {
    bool public opened;

    function open(address guardian) external {
        uint256 codeSize;
        assembly {
            codeSize := extcodesize(guardian)
        }
        require(codeSize < 33, "bad code size");

        require(
            IGuardian(guardian).f00000000_bvvvdlt() == address(this),
            "invalid pass"
        );
        require(
            IGuardian(guardian).f00000001_grffjzz() == tx.origin,
            "invalid pass"
        );

        (bool success, ) = guardian.call(abi.encodeWithSignature("fail()"));
        require(!success);

        opened = true;
    }
}
```